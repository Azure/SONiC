# Introduction

The scope of this document is to provide the requirements and a high-level design proposal for VNET monitoring.

# Requirements

The following are the high level requirements for the VNET monitoring infrastructure:
1. Any arbitrary pair of source ToR (SToR) and destination ToR (DToR) shall be able to send and recieve a data from and to VNET HW pipeline without using connected hosts.
2. The SToR shall be able to inject an overlay packet from a CPU port into the HW pipeline for further encapsulation and underlay routing.
3. The DToR shall be able to detect the injected packet after decapsulation and redirect it back to the SToR.
4. The user interface to the monitoring tool is CLI based, e.g.:
```
vnet-ping <vnet_name> <vlan_intf|optional> <ca_src_ip|optional> <ca_dst_ip>
```
5. Required input to the tool is:
   * Source VNET name to inject packet to
   * Optional VLAN interface name that belongs to `vnet_name`
   * Optional BM src IP
   * BM dst IP
6. Connectivity is checked only in one direction from SToR to DToR. The way back should be checked from DToR.

# SToR

The function of the SToR is to inject an overlay packet with the cookie for the DToR to trap and redirect back.
The cookie must be known to DToR ahead of time.

# DToR

The function of the DToR is to detect a packet from SToR by the cookie, swap underlay SIP and DIP, and send it back to SToR

## Kernel configuration

For each VLAN interface, that is a member of the VNET, a kernel netdev representation is created.

```
SAI_HOSTIF_ATTR_TYPE = SAI_HOSTIF_TYPE_NETDEV;
SAI_HOSTIF_ATTR_OBJ_ID = port.m_vlan_info.vlan_oid;
SAI_HOSTIF_ATTR_NAME = <hostif_name>;
```

An optional field to indicate the need to create a host interface is needed in the VLAN schema:

```
    "VLAN": {
        "Vlan2000": {
            "vlanid": 2000,
            "host_ifname": "MonVlan2000"

        }
    },
```

## Supported interfaces

The VNET monitor supports packet injection into VLAN router interfaces.

# Approaches for identifying the injected packets

The packets injected by a ping tool need to be identified by the data plane and lifted to the CPU.

The general approach to doing so is installing a dedicated ACL rule that would match on the packet's coockie and send a packet to a user defined trap.
It means that SONiC ACL needs to be extended with a new table for such use.

An intermediate solution will be to use the standard traps, like "TTL check". It won't require the additional ACL installation, but is based of some assumptions:
* The VxLAN tunnel TTL mode is set to "pipe"
* There is no Linux VxLAN device
* TTL trap action is set to "trap"

Below are the two packet flows showing the packet processing depending on the approach taken. The SToR packet processing is pretty much the same, except in case of using TTL trap ping tool will explicitly set the TTL value in the injected packet to 2.

# SToR packet walkthrough (ACL based)

![](https://github.com/marian-pritsak/sonic/blob/patch-2/doc/vxlan/SToR_ACL.png)

1. inject packet to VLAN ID under source VNET
2. packet is classified to VNET VRF by VLAN RIF
3. overlay router looks up tunnel next hop
4. VxLAN tunnel encap
5. underlay routing to DToR

# SToR packet walkthrough (TTL based)

![](https://github.com/marian-pritsak/sonic/blob/patch-2/doc/vxlan/SToR.png)

1. inject packet to VLAN ID under source VNET
2. packet is classified to VNET VRF by VLAN RIF
3. overlay router looks up tunnel next hop
4. TTL is decremented to 1
5. VxLAN tunnel encap
6. underlay routing to DToR

# DToR packet walkthrough (ACL based)

![](https://github.com/marian-pritsak/sonic/blob/patch-2/doc/vxlan/DToR_ACL.png)

1. underlay routing
2. VxLAN decapsulation
3. packet is classified to VRF by VNI
4. overlay routing
5. overlay bridging
5. ACL user defined trap
6. packet redirection by vnet-pingd

# DToR packet walkthrough (TTL based)

![](https://github.com/marian-pritsak/sonic/blob/patch-2/doc/vxlan/DToR.png)

1. underlay routing
2. VxLAN decapsulation
3. packet is classified to VRF by VNI
4. overlay routing
5. "ttl too small" exception
6. packet redirection by vnet-pingd
