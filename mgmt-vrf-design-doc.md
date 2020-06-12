Management VRF Design Document

Introduction
------------

Management VRF is a subset of VRF (virtual routing tables and forwarding) and provides a separation between the out-of-band management network and the in-band data plane network. For all VRFs, the main routing table is the default table for all of the data plane switch ports. With management VRF, a second table, mgmt, is used for routing through the Ethernet ports of the switch. The following design for mgmt-vrf uses the l3mdev approach for implementating mgmt-vrf on SONiC.

Requirements
------------
                                              Table - 1
    | Req No | Description                                                                                                | Priority | Comments |
    |--------|------------------------------------------------------------------------------------------------------------|----------|----------|
    | 1      | Develop and implement a separate Management VRF that provide management plane and Data Plane Isolation     |          |          |
    | 2      | Management VRF should be associated with a separate L3 Routing table and the management interface          |          |          |
    | 3      | Management VRF and Default VRF should support IP services like Ping and traceroute  in its context         |          |          |
    | 4      | Management VRF should provide the ability to be polled via SNMP both via default VRF and management VRF    |          |          |
    | 5      | Management VRF will provide the ability to send SNMP traps both via the   management VRF and data network. |          |          |
    | 6      | Enable DHCP services to run in Management VRF context                                                      |          |          |
    |        | ˇ       Dhcpd                                                                                              |          |          |
    |        | ˇ       Dhcrelay  Default VRF support                                                                     |          |          |
    |        | ˇ       dhclient                                                                                           |          |          |
    | 7      | Enable SSH services to run in the Management VRF context and Default VRF context                           |          | 
    | 8      | Enable TFTP services to run in the Management VRF context and Default VRF context                          |          |
    | 9      | Management VRF should support NTP services in its context and default VRF context                          |          |          |
    | 10     | Management VRF and Default VRF should support wget. Curl and HTTPS services                                |          |          |
    | 11     | Management VRF and Default VRF should support apt-get package managers in the Management VRF context       |          |          |
    | 12     | Management VRF will provide TACACS+ support.                                                               | Low      |          |
   

Use-Cases
---------

     1.Out of band management
      ----------------------

      Only Management port is part of management VRF; No FEP is part of management VRF

                                              Dest IP = Mgmt. IP              	              Dest IP ≠ Mgmt. IP
                                             ------------------                               ------------------
      Packet coming via mgmt. port    	  All IP services listed in Table 1 should          Packet dropped. (No one arm routing?)
                                          be supported  
 
      Packet coming via Front end port    Treat the packet like any other data traffic.     Treat the packet like any other data traffic.
                                          To be sent out as per the routing table lookup    To be sent out as per the routing table lookup
                                          Dest IP = FEP IP in Default VRF                   Dest IP ≠ FEP IP in Default VRF

      Packet coming via mgmt. port        Packet dropped                                    Packet dropped

      Packet coming via Front end port	  All IP services listed in Table 1 should          Treat the packet like any other data traffic. 
                                          be supported.                                     To be sent out as per the routing table lookup

     2.In band management
      ------------------

      Only Front-end port is part of management VRF; No management port connectivity (typically)

                                             Dest IP = Mgmt. IP                              Dest IP ≠ Mgmt. IP
                                             ------------------                              ------------------
      Packet coming via mgmt. port        Not applicable                                   Not applicable

      Packet coming via Front end port	  All IP services listed in Table 1 should         (One arm routing supported? Need to 
                                          be supported                                      test in FTOS/OS10)
                                          Dest IP = Front end port IP                      Dest IP ≠ Front end port IP

     Packet coming via mgmt. port         Not applicable                                   Not applicable

     Packet coming via Front end port	  All IP services listed in Table 1 should    	   Treat the packet like any other data traffic.
                                          be supported                                     To be sent out as per the routing table lookup
                                                                                           in management VRF
     3.Switch originated traffic
      -------------------------

      Unless explicitly specified to use mamangment VRF, all the services will use the default routing table to egress out of switch.

Design Approach
---------------

    | Features         | Approach  Name space            | Approach  L3MDev                             |
    |------------------|----------------------------------|-----------------------------------------------|
    | Kernel Support   | Yes                              | Yes                                           |
    | Scalability      | Less scalable                    | Better Scalable                               |
    | Protocol support | IP services should be replicated | IP services can be enslaved to a L3 interface |
    | Performance      | Overhead  of replicated services | Shared services                               |

Network namespaces were designed to provide a complete isolation of the entire network stack —devices, network addresses, neighbor and routing tables, protocol ports and sockets. 
VRF On the other hand is a network layer 3 feature and as such should really only impact FIB lookups. While the isolation provided by a namespace includes the route tables, a network namespace will have  a significant overhead as we will end up having two network stacks.

Using network name space as VRF’s will involve having services replicated across all VRF’s – for example we will have run two instances of lldp/ssh for Management and Default VRF’s respectively. 
Also in terms of extensibility, using namespaces for Multi VRF means N-VRF’s which will have N-application instances and also the additional complexity of aggregating all the data across these instances.
 
With L3-Mdev on 4.9 , VRF’s are created and enslaved to an L3 Interface and services like ssh can be shared across VRF’s thereby avoiding overhead of running multiple instances of IP services.


Phase-1
-------

In this Phase mgmt-vrf feature will be enabled by default when SONiC boots with Debian 4.9 kernel, configuration of mgmt-vrf will be
done using linux commands.


DB Schema
---------

The config_db.json schema is updated to have the vrfname in MGMT_INTERFACE. The schema representation will look like below

    "MGMT_INTERFACE": {
        "eth0|10.11.150.19/24": {
            "gwaddr": "10.11.150.254"
            "vrfname": "mgmt-vrf"
         }
    }

The vrfname if set to mgmt-vrf will program the linux kernel with the vrf configuration and create a mgmt-vrf for the mgmt
port eth0. If it is set to "None" then the mgmt-vrf will not be configured on the system. Management VRF will be enabled by default when SONiC boots. 


Flow
-----

By default, mgmt-vrf will be configured in Debian 4.9 kernel, when SONiC boots there will be isolation of traffic between
mgmt traffic and data traffic. This is acheived by setting the vrfname to mgmt-vrf in the minigraph.py file as default setting.
The interfaces.j2 file checks for the vrfname tag to see if it is set to mgmt-vrf. If the vrfname is set then it creates the
mgmt-vrf configuration by executing the lniux commands. Debian 4.9 kernel does not support running services per VRF, by enabling
the tcp_l3mdev_accept=1 the services will work across all VRF's. Below diagram shows the flows of event for creating mgmt-vrf on
SONiC using Debian 4.9

![](https://i.imgur.com/Ge6SNX4.png)


Phase-2
-------

Later when kernel is upgraded to 4.15 or later support for running services in separate VRF will be available. New CLI command will be
provided to create and delete VRF using config commands. Config-save command will be used to save the configuration to config_db.

Proposed New CLI
----------------

The following CLI can be used to Configure and show vrf's.

    config vrf add/del <vrfName>
    config vrf member add/del <vrfName> <interfaceName>
    show vrf config
    show vrf brief
    show vrf <vrfname>



Flow
-----

The following modules will be affected in phase-2 for management VRF configuration. 

    swss
     - cfgmgr
       - vrfmgrd.cpp
       - vrfmgr.cpp

Configure vrf using the config cli above, this triggers the vrfmgrd to create/delete the vrf in linux. Changes to configuration files for services is required for services to run per VRF instance.


mgmt-vrf Configuration Commands
-------------------------------
    # management vrf configuration
    1. Create a VRF
       ip link add name <vrfname> type vrf table <id>

    2. Set the vrf to up
       ip link set dev <vrfname> up

    3. Assign a Network Interface to a VRF
       // Network interfaces are assigned to VRF by enslaving the netdevice to VRF device.
       ip link set dev eth0 master <vrfname>

    4. Add a route to the mgmt-vrf
       ip route add vrf <vrfname> 0.0.0.0/0 via <gwaddr>

mgmt-vrf Linux Show Commands
----------------------------
    show commands 
    ------------- 
    1. Default VRF show command
       ip route show

    2. mgmt-vrf show command
       //Table id is the id used in create VRF command above.
       ip route show table <id> 

    3. List's all the VRF that are created. 
       // -d option shows the table number or id
       ip -d link show vrf 

    4. show command to dump brief output of vrf.
       ip -br link show type vrf

    5. show command to dump vrf output
       ip link show type vrf

    6. show command to dump vrf addr
       ip addr show vrf <vrfname>

mgmt-vrf Show Command (SONiC Wrapper)
---------------------

    | SONiC Utility command           | Linux command                  | Description                          |
    |---------------------------------|--------------------------------|--------------------------------------|
    | show vrf                        | ip link show type vrf          | Display the VRF's configured         |
    | show vrf  <vrfname> brief       | ip  -br link show type vrf     | Display VRF brief info               |
    | show vrf  <vrfname> detail      | ip -d link show type vrf       | Displays VRF detailed info           |
    | show vrf route                  | ip route show                  | Displays the default VRF routes      |
    | show vrf route table <table-id> | ip route show table <table-id> | Displays the VRF routes for table-id |
    | show vrf address  <vrfname>     | ip address show vrf <vrfname>  | Displays IP related info for VRF     |

Utilities and Tools
-------------------
    1. ping commands
       // default VRF ping command
       ping x.x.x.x
   
       //mgmt-vrf ping command
       ping -I mgmt-vrf x.x.x.x

    2. Traceroute commands
       // default VRF traceroute command
       traceroute x.x.x.x

       //mgmt-vrf traceroute command
       traceroute -i x.x.x.x

Example Output Of Commands
--------------------------
    root@sonic:~# ip route show
    1.1.1.0/24 dev Ethernet4 proto kernel scope link src 1.1.1.2
    2.1.1.0/24 dev Ethernet5 proto kernel scope link src 2.1.1.2
    10.0.0.0/31 dev PortChannel1 proto kernel scope link src 10.0.0.0 linkdown
    10.0.0.4/31 dev PortChannel5 proto kernel scope link src 10.0.0.4 linkdown
    10.0.0.8/31 dev PortChannel16 proto kernel scope link src 10.0.0.8 linkdown
    10.0.0.12/31 dev PortChannel20 proto kernel scope link src 10.0.0.12 linkdown
    172.0.0.0/26 dev Vlan2 proto kernel scope link src 172.0.0.1 linkdown
    240.127.1.0/24 dev docker0 proto kernel scope link src 240.127.1.1 linkdown
    root@sonic:~#
    root@sonic:~# ip route show table 1
    default via 10.11.55.254 dev eth0
    broadcast 10.11.55.0 dev eth0 proto kernel scope link src 10.11.55.38
    10.11.55.0/24 dev eth0 proto kernel scope link src 10.11.55.38
    local 10.11.55.38 dev eth0 proto kernel scope host src 10.11.55.38
    broadcast 10.11.55.255 dev eth0 proto kernel scope link src 10.11.55.38
    root@sonic:~#
    root@sonic:~# 
    root@sonic:~# ip -d link show type vrf
    145: mgmt-vrf: <NOARP,MASTER,UP,LOWER_UP> mtu 65536 qdisc noqueue state UP mode DEFAULT group default qlen 1000
        link/ether 0e:bb:43:3e:18:ae brd ff:ff:ff:ff:ff:ff promiscuity 0
        vrf table 1 addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535
    root@sonic:~#
    root@sonic:~#
    root@sonic:~# ip link show type vrf
    145: mgmt-vrf: <NOARP,MASTER,UP,LOWER_UP> mtu 65536 qdisc noqueue state UP mode DEFAULT group default qlen 1000
        link/ether 0e:bb:43:3e:18:ae brd ff:ff:ff:ff:ff:ff
    root@sonic:~#
    root@sonic:~#
    root@sonic:~# 
    root@sonic:~# ip -br link show type vrf
    mgmt-vrf         UP             0e:bb:43:3e:18:ae <NOARP,MASTER,UP,LOWER_UP>
    root@sonic:~#
    root@sonic:~#

    root@sonic:~# ip addr show vrf mgmt-vrf
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master mgmt-vrf state UP group default qlen 1000
        link/ether 4c:76:25:f5:54:80 brd ff:ff:ff:ff:ff:ff
        inet 10.11.55.38/24 brd 10.11.55.255 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::4e76:25ff:fef5:5480/64 scope link
           valid_lft forever preferred_lft forever
    root@sonic:~# ip addr show type vrf
    145: mgmt-vrf: <NOARP,MASTER,UP,LOWER_UP> mtu 65536 qdisc noqueue state UP group default qlen 1000
        link/ether 0e:bb:43:3e:18:ae brd ff:ff:ff:ff:ff:ff
    root@sonic:~#
    root@sonic:~# ping -I mgmt-vrf 8.8.8.8
    ping: Warning: source address might be selected on device other than mgmt-vrf.
    PING 8.8.8.8 (8.8.8.8) from 10.11.55.38 mgmt-vrf: 56(84) bytes of data.
    64 bytes from 8.8.8.8: icmp_seq=1 ttl=119 time=3.55 ms
    64 bytes from 8.8.8.8: icmp_seq=2 ttl=119 time=2.77 ms

    --- 8.8.8.8 ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 1001ms
    rtt min/avg/max/mdev = 2.772/3.165/3.559/0.397 ms

    root@sonic:~# traceroute -i mgmt-vrf  8.8.8.8
    traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
     1  10.11.55.254 (10.11.55.254)  0.468 ms  0.441 ms  0.468 ms
     2  10.11.3.254 (10.11.3.254)  0.599 ms 10.11.3.253 (10.11.3.253)  0.885 ms 10.11.3.254 (10.11.3.254)  0.886 ms
     3  10.11.27.254 (10.11.27.254)  0.521 ms  0.482 ms  0.798 ms
     4  63.80.56.65 (63.80.56.65)  1.141 ms  1.278 ms  2.977 ms
     5  65.198.30.33 (65.198.30.33)  1.982 ms  2.871 ms  1.999 ms
     6  152.179.99.173 (152.179.99.173)  3.307 ms  2.442 ms  4.390 ms
     7  140.222.237.207 (140.222.237.207)  6.141 ms 140.222.237.205 (140.222.237.205)  8.052 ms 140.222.237.207 (140.222.237.207)  6.091 ms
     8  152.179.48.74 (152.179.48.74)  4.229 ms  2.930 ms  5.132 ms
     9  108.170.242.81 (108.170.242.81)  3.316 ms 108.170.243.1 (108.170.243.1)  7.184 ms  5.406 ms
    10  216.239.46.167 (216.239.46.167)  6.463 ms 74.125.37.41 (74.125.37.41)  4.269 ms 108.170.230.87 (108.170.230.87)  6.593 ms
    11  8.8.8.8 (8.8.8.8)  4.861 ms  5.541 ms  4.087 ms



Future Changes For Linux Upgrade
--------------------------------

    Minimal changes required to support multiple services or configuration files across multiple VRF's. Can leverage the latest feature that are 
    supported as we upgrade Linux.

    Changes will be required to run IP services per VRF, we will vist this and update the design accordingly in future.

    ip vrf exec command will be available in the next linux upgrade when iproute2 utilities will also be upgraded.

