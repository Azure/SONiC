# Vxlan SONiC
# fpmsyncd design
### Rev 0.1

# Table of Contents
  * [List of Tables](#list-of-tables)

  * [Revision](#revision)

  * [Scope](#scope)

  * [1 Requirements](#1-requirements)

  * [2 Flows](#2-flows)
    * [2.1 Identify VNet routes](#21-identify-vnet-routes)
    * [2.2 Parse VNet routes](#22-parse-vnet-routes)
      * [2.2.1 Identify VNet regular and tunnel routes](#221-identify-vnet-regular-and-tunnel-routes)
      * [2.2.2 Handle VNet regular and tunnel routes](#222-handle-vnet-regular-and-tunnel-routes)
  * [3 Reference Tables](#3-reference-tables)

  * [4 Examples](#4-examples)

###### Revision
| Rev |     Date    |       Author       | Change Description                |
|:---:|:-----------:|:------------------:|-----------------------------------|
| 0.1 |      04/21/2019       |     Wei Bai   | Initial version                   |

# Scope
This document describes the design of fpmsyncd to support VNet routes. 

# Definitions/Abbreviation
###### Table 1: Abbreviations
|                          |                                |
|--------------------------|--------------------------------|
| VRF                      | Virtual Routing and Forwarding |
| VNet                     | Virtual Network                |
| Vxlan                    | Virtual Extensible Local Area Network |
| VTEP                     | Vxlan Tunnel End Point         |

# 1 Requirements
This section describes the SONiC requirements for fpmsyncd in the context of VNet.

At a high level the following features should be supported:

Phase #1
- Identify VNet routes from all the receiving routes.
- Parse VNet routes to insert/delete the right entries into/from the App DB.

Phase #2
- Support warm restart for VNet routes.

# 2 Flows
## 2.1 Identify VNet routes
Given a input route, fpmsyncd first uses <code>rtnl_route_get_table</code> to get the master device's interface ID. Then fpmsyncd uses this interface ID to derive the name of the master device.  

fpmsyncd identifies VNet routes based on the name of the master device. If the name of the master device starts with <code>"Vnet"</code>, the route is a VNet route. 

## 2.2 Parse VNet routes
After identifying VNet routes, the next challenge is how to parse them correctly to insert/delete the right entries into/from the App DB.

### 2.2.1 Identify VNet regular and tunnel routes
There are two types of VNet routes: regular routes and tunnel routes. Regular routes just forward packets to the outgoing interfaces as usual. In contrast, tunnel routes first encapsulate packets with Vxlan headers and then forwards packets based on the remote VTEP's IP address. The above two types of VNet routes are handled by different tables on the App DB.

We leverage the route's outgoing interfaces' name to differentiate VNet regular routes and VNet tunnel routes. If the first interface name starts with <code>"Brvxlan"</code>, the route is a VNet tunnel route. Otherwise, it is a regular VNet route. Note that <code>"Brvxlan"</code> is the prefix of SONiC Vxlan interface name. For more details, please refer to the implantation of Vxlanmgr in SONiC swss. 

### 2.2.2 Handle VNet regular and tunnel routes
For VNet regular routes, their information is inserted/deleted into/from <code>VNET_ROUTE_TABLE</code> of App DB.

For VNet tunnel routes, their information is inserted/deleted into/from <code>VNET_ROUTE_TUNNEL_TABLE</code> of App DB.

# 3 Reference Tables
```
VNET_ROUTE_TABLE:{{vnet_name}}:{{prefix}} 
    "nexthop": {{ip_address}} (OPTIONAL) 
    "ifname": {{intf_name}} 
 
VNET_ROUTE_TUNNEL_TABLE:{{vnet_name}}:{{prefix}} 
    "endpoint": {{ip_address}} 
    "mac_address":{{mac_address}} (OPTIONAL) 
    "vni": {{vni}}(OPTIONAL) 
```

# 4 Examples
Assume that we have a VNet regular route. The VNet name is <code>"Vnet1"</code>. The destination IP prefix is <code>"192.168.1.0/24"</code>. The next hop IP address is <code>"172.16.0.1"</code>. The interface name is <code>"Ethernet0"</code>. The App DB object of this route is given as follows:
```
"VNET_ROUTE_TABLE:Vnet1:192.168.1.0/24" 
    "nexthop": "172.16.0.1"
    "ifname": "Ethernet0"
```

Assume that we have a VNet tunnel route. The VNet name is <code>"Vnet1"</code>. The destination IP prefix is <code>"192.168.1.0/24"</code>. The next hop IP address is <code>"172.16.0.1"</code>. The interface name is <code>"Brvxlan8000"</code>. Given the special inteface name <code>"Brvxlan8000"</code>, we know that this is a VNet tunnel route. The App DB object of this route is given as follows:
```
"VNET_ROUTE_TUNNEL_TABLE:Vnet1:192.168.1.0/24"
    "endpoint": "172.16.0.1"
```   



