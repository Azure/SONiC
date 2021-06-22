# Routing Warm Reboot with Dynamic Reconciliation Signal 


## Problem Definition:
In current implementation of SONiC routing warm reboot or warm restart, there is a waiting time between fpmsyncd initialization and fpmsyncd to AppDB reconciliation. Right now the time is set to a pre-defined value (120s by default). This could slow down forwarding path convergence during warm reboot, because there is a hard requirement on the time window size - it has to be large enough to ensure the completion of all initial routing updates. To meet this requirement, the time slot is usually assigned to a value larger than what is actual needed. 

In order to minimize the waiting time, while not missing any initial updates, a more accurate way to start the fpmsyncd to AppDB reconciliation would be aligning forwarding path updates dynamically with upper layer routing protocols (particularly BGP's for now) convergence time.

An End-of-Routing(EOR) scheme for BGP routing update convergence has already been defined in IETF RFC4724. Based on that, current FRR BGP implementation is meant to hold back all BGP updates in its work queue, until all necessary EOR markers are received, or maximum waiting time expires. By then, BGP creates an End-of-Initial Update (EOIU) signal. BGP keeps on checking the EOIU signal. Once the signal is set, BGP immediately unplugs its work queue, which in turn triggers RIB update in Zebra. However, this EOIU signal is currently not passed to Zebra nor fpmsyncd.

## Proposed Solution:
This proposal defines a way to allow the passing of the EOIU signal from BGP -> Zebra -> Zfpm -> fpmsyncd. Instead of waiting for a fixed timer to expire, fpmsync can start route reconciliation with AppDB based on EOIU signal, which indicates the completion of upper layer routing convergence. 

#### EOIU entry queued in FIFO 
To minimize the changes in the routing code and hence the risk of disrupting existing functionality, BGP EOIU signal is created as a normal routing entry, with dummy content, and a newly defined routing type in Zebra as
``` ZEBRA_ROUTE_BGP_EOIU ```
and a new rtm\_protocol type in netlink as 
``` RTPROT_EOIU ```
. The dummy entry is passed from BGP all the way down to fpmsyncd in FIFO queues. In fpmsyncd, upon receiving a routing entry with RTPROT\_EOIU, as well as corresponding AFI, from netlink, the routing reconciliation to AppDB starts. 

This is the main path to handle EOIU in Zebra.

#### EOIU flag set in RIB table
During BGP reboot, Zebra to fpm connection could take longer time than routing protocol convergence. When this happens, all routing entries, including the one that carries the EOIU signal in the zfpm queue, will be released and can't be passed to fpmsyncd. Later, when the fpm connection comes up, Zebra will resend all routing entries to fpm, by walking its table in radix tree.

In order to mark the EOIU signal in Zebra table, an EOIU flag is added to Zebra to indicate the receiving of EOIU from BGP. When fpmsyncd connection is up and if the flag is set, Zebra will generate an EOIU style dummy route entry to fpmsyncd at the end of its table walk. 

#### Notes
* We still keep the existing warmRestartIval in fpmsyncd as the safety net in case the EOIU signal is not received, even though this should not happen in current FRR implementation. 

* Besides AFI, ideally SAFI, or even a more generic tableID, to cover routing hierarchy, such as VRFs, topologies etc, could be passed to fpmsyncd, to indicate EOIU for a particular routing table. But there is no immediate deployment request for it, hence we decided to leave it to next phase (also refer to the section of Limitations for more details).



## Detailed Code Flow:

#### Step 1: BGP to Zebra
Upon EOIU being processed, BGP passes the signal to RIB. This is done by creating a dummy route, with a new ZEBRA\_ROUTE\_BGP\_EOIU type, and the AF, which matches the one specified in BGP work queue.

```
bgp_process_wq
  bgp_process_main_one
    if (!rn)  {
    	bgp_zebra_announce_table 
    	+ dummy_addr.family = afi2family(afi);
    	+ bgp_zebra_announce(NULL, &dummy_addr, NULL, bgp, afi,  safi);
    	...
    	}
```

#### Step 2: Zebra
Zebra process the dummy route in normal path, but some special care has to be taken in Zebra, to secure the pass through of the dummy route:

* It should not be dropped by various sanity checks, such as next hop validation, etc.
* It should be inserted into queues with proper priority and order, so that it can be always processed after all initial routes. 
* It should not be inserted into routing table.

As we discussed early, an EOIU flag should also be set in Zebra, in case routing protocols are converged, but fpm connection is not up yet. Since all routing update messages in zfpm queue are freed when a down connection is detected , the messages have to be re-composed later when fpm connection is up, where the EOIU flag saved in Zebra is used to re-generate a dummy routing entry to indicate EOIU. 

```
zebra_event 
  zebra_client_read 
    zserv_handle_commands
      case ZEBRA_ROUTE_ADD:
        zread_route_add
	  re = XCALLOC(MTYPE_RE, sizeof(struct route_entry));
	  STREAM_GETC(s, re->type); <============ ZEBRA_ROUTE_BGP_EOIU
	  afi = family2afi(api.prefix.family); <--- get afi from prefix
	  rib_add_multipath
	  + if (re->type == ZEBRA_ROUTE_BGP_EOIU) {
	  +     rn = rib_add_eoiu(afi, safi, p, src_p, table);
	  + } else {
	  	     rn = srcdest_rnode_get(table, p, src_p);
	  + }
	    rib_addnode
	      rib_link
	        rib_queue_add
		  rib_meta_queue_add(struct meta_queue *mq, struct route_node *rn)
		    listnode_add(mq->subq[qindex], rn);  <--- priority 3
```

- Zebra to zfpm -

```
meta_queue_process 
  process_subq(struct list *subq, u_char qindex)
    lnode = listhead(subq);
    rnode = listgetdata(lnode);
    rib_process(struct route_node *rn)
      rib_process_update_fib rib_process_add_fib...
        hook_call(rib_update, rn,...) = invoke zfpm_trigger_update
	       TAILQ_INSERT_TAIL(&zfpm_g->dest_q, dest, fpm_q_entries); <******
	         zfpm_write_on ();

```

#### Step 3: Zebra to FPM
Convert the RIB entry into netlink format. A new RTPROT_EOIU is defined in FRR, and hard coded in fpmsyncd, to avoid linux include file changes in rtnetlink.h.  

```
zfpm_write_cb
  zfpm_build_updates
    dest = TAILQ_FIRST (&zfpm_g->dest_q); <*******
    zfpm_route_for_update
      CHECK_FLAG (rib->flags, ZEBRA_FLAG_SELECTED)
    zfpm_encode_route
      switch (zfpm_g->message_format) {
        *msg_type = FPM_MSG_TYPE_NETLINK
      zfpm_netlink_encode_route
        netlink_route_info_fill
	       ri->rtm_protocol = netlink_proto_from_route_type(re->type);
	       ri->rtm_protocol = RTPROT_EOIU; <====== add a new type for eoiu
	  ...
        netlink_route_info_encode
   + zfpm_route_eoiu_reset
```

#### Step 4: fpmsyncd
For a routing entry received in fpmsyncd, decode its protocol type. If the type is RTPROT_EOIU, retrieve the AF. Now the fpmsyncd can use the AF specific signal to kick off the reconciliation process for routes in a particular AF.

```
RouteSync::onMsg
	 if (rtnl_route_get_protocol(route_obj) == RTPROT_EOIU) {
```


## Limitations:

#### 1) BGP EIOU marker is not AFI/SAFI specific

FRR BGP implementation doesn't handle EOR signal independently for each AFI/SAFI as described by IETF RFC4724. EOIU marker currently is set based on
BGP neighbors, instead of on each AF/SAFI of neighbors.  

```
bgp_update_explicit_eors
	for (afi = AFI_IP; afi < AFI_MAX; afi++)  <==== check all AFI/SAFI
		for (safi = SAFI_UNICAST; safi < SAFI_MAX; safi++) {
			if (peer->afc_nego[afi][safi]
			    && !CHECK_FLAG(peer->af_sflags[afi][safi],
					   PEER_STATUS_EOR_RECEIVED)) {
				return;
			}
		}


bgp_process_wq
  if (CHECK_FLAG(pqnode->flags, BGP_PROCESS_QUEUE_EOIU_MARKER)) {
      bgp_process_main_one(bgp, NULL, 0, 0); <==== pass in 0/0 as AFI/SAFI
}

bgp_process_main_one(struct bgp *bgp, struct bgp_node *rn,
			           afi_t afi, safi_t safi)
{
	if (!rn) {
		FOREACH_AFI_SAFI (afi, safi) {   <==== loop all
			if (bgp_fibupd_safi(safi))
				bgp_zebra_announce_table(bgp, afi, safi);
		}
		return;
	}

```

We won't change above BGP EOIU creation logic in current release, which means though the EOIU signal passed down to Zebra is AFI/SAFI specific, the origination of the EOIU signal is not.


#### 2) EIOU signal pushed down to FPM carries only AFI, not SAFI, nor VPN.

```
bgp_zebra_announce 
  api.safi = safi;
  api.prefix = *p; 
  
zebra_event 
  zebra_client_read 
	  afi = family2afi(api.prefix.family); <------ get afi from prefix
	  rib_add_multipath(afi, api.safi, &api.prefix, src_p, re);
	    table = zebra_vrf_table_with_table_id(afi, safi, re->vrf_id, re->table); <-- afi/safi to table
	    
zfpm_write_cb
  zfpm_build_updates
    zfpm_encode_route
      zfpm_netlink_encode_route
        netlink_route_info_fill
          ri->af = rib_dest_af(dest);  <------ 

RouteSync::onMsg
  auto family = rtnl_route_get_family(route_obj); 
  <---- no xxx_get_sub_family
  <---- rtnl_route_get_table () not used
  
```
We currently have only IPv4 and IPv6 unicast supported in SoNIC. If multicast, VPN or multi-topology is required in the future, above logic needs to be extended, and a more generic tableID might be carried.

#### 3) EOIU is interpreted only in fpmsyncd

The RTPROT_EOIU inserted as a new netlink rtm_protocol is only decoded in fpmsyncd, but not in kernel.  

#### 4) EOIU is set for BGP only

Ideally the EOIU signal in Zebra should be an indicator of routing convergence from all protocols, not only BGP. We will address other routing protocols in later release if needed.

[Author]  Heidi Ou,  Nikos Triantafillis
