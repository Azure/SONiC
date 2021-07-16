# SONiC Weighted ECMP High Level Design Document #
#### Rev 0.1

## Table of Contents
- [SONiC Weighted ECMP High Level Design Document](#sonic-weighted-ecmp-high-level-design-document)
      - [Rev 0.1](#rev-01)
  - [Table of Contents](#table-of-contents)
    - [Revision](#revision)
    - [Scope](#scope)
    - [Definitions/Abbreviations](#definitionsabbreviations)
    - [Requirements](#requirements)
      - [Functional requirements](#functional-requirements)
      - [Scalability Requirements](#scalability-requirements)
      - [Future Requirements](#future-requirements)
    - [Architecture Design](#architecture-design)
    - [High-Level Design](#high-level-design)
      - [Overview](#overview)
      - [Database Changes](#database-changes)
        - [ROUTE_TABLE](#route_table)
      - [Software Modules](#software-modules)
        - [FPM Syncd](#fpm-syncd)
          - [fpmsyncd functions](#fpmsyncd-functions)
        - [RouteOrch](#routeorch)
          - [routeorch functions](#routeorch-functions)
        - [NextHopKey](#nexthopkey)
    - [SAI API](#sai-api)
    - [Configuration and management](#configuration-and-management)
      - [CLI Enhancements](#cli-enhancements)
      - [Config DB Enhancements](#config-db-enhancements)
    - [Warmboot and Fastboot Design Impact](#warmboot-and-fastboot-design-impact)
    - [Restrictions/Limitations](#restrictionslimitations)
    - [Testing Requirements/Design](#testing-requirementsdesign)
      - [Unit Test cases](#unit-test-cases)
      - [System Test cases](#system-test-cases)
    - [Open/Action items](#openaction-items)
### Revision
|  Rev  |    Date    | Author | Change Description |
| :---: | :--------: | :----: | ------------------ |
|  0.1  | 01/20/2021 | Z Cai  | Initial version    |

### Scope
This document provides general information about the Weighted ECMP feature implementation in SONiC. This design is focusing on implementation at SWSS layer and below. The configuration/process in routing stack which sends different weight value in Netlink messages is not covered here.

### Definitions/Abbreviations
| Abbreviation | Description           |
| ------------ | --------------------- |
| ECMP         | Equal Cost Multi-Path |

### Requirements
This section describes the SONiC requirements for:
- Weighted ECMP routes feature.

#### Functional requirements
- Support for weighted ECMP routes for IPv4/IPv6.

#### Scalability Requirements
- Up to max ASIC capable ecmp routes are supported.

#### Future Requirements
- Support for weighted ECMP label routes for MPLS.
- Integration with routing stack.

### Architecture Design
With introduction of weighted ECMP, there is no architecture change to existing SONiC

### High-Level Design
The weighted ECMP feature extends Next Hop Group support in SONiC to include optional weight in addition to the existing Next Hop Group Member information.

#### Overview
SONiC currently supports ECMP(Equal Cost MultiPath) for Next Hop Group Members. This means traffic for the destination is distributed equally among each Next Hop Group Members.
However, when Next Hop Group Members have different capacity(e.g. Link speed, AE bundle size etc), ECMP would create inbalanced link utilization among members. Weighted ECMP enables load balancing of traffic between equal cost paths in proportion to the capacity of the Next Hop Group Members.

#### Database Changes
This section describes the modifications to SONiC Databases to support Weighted ECMP routes.

The existing ROUTE_TABLE in APPL_DB for IPv4/IPv6 prefix routes is enhanced to accept an optional new "weight" attribute.

##### ROUTE_TABLE
``` rfc5234
"ROUTE_TABLE":{{prefix}}
    "nexthop":{{nexthop_list}}
    "ifname":{{ifname_list}}
    "weight":{{weight_list}} (OPTIONAL)

; Defines schema for IPv4/IPv6 route table attributes
key                         = ROUTE_TABLE:prefix       ; IPv4/IPv6 prefix
; field                     = value
nexthop                     = nexthop_list             ; List of nexthops.
ifname                      = ifname_list              ; List of interfaces.
weight                      = weight_list              ; List of weights.
;value annotations
ifname                      = 1*64VCHAR                ; name of the Interface (Port Channel)
```
#### Software Modules
This section describes modifications to software modules to support Weighted ECMP routes.

##### FPM Syncd
FPM Syncd is an existing daemon in BGP container that monitors NetLink messages from the SONiC routing stack.
FPM Syncd now additionally processes Next Hop information in the NetLink messages and sets this information in the APPL DB.

###### fpmsyncd functions
The following are new functions for fpmsyncd:
```
  /* Handler to process ECMP Weight information rtnl messages */
  string RouteSync::getNextHopWt(struct rtnl_route *route_obj);
```
##### RouteOrch
RouteOrch is an existing component of the OrchAgent daemon in the SWSS container.  RouteOrch monitors operations on Route related tables in APPL DB and converts those operations in SAI commands to manage Route entries.  Additionally RouteOrch coordinates Next Hop object operations with NeighOrch and converts operations into SAI commands to manage Next Hop Group objects.

###### routeorch functions
The following functions are modified to support weight field in Next Hop Group Members.
```
  /* Handler to process ECMP Weight information from APPL DB */
  void RouteOrch::doPrefixTask(Consumer& consumer)
  /* Handler to call SAI API with Weight information */
  bool RouteOrch::validnexthopinNextHopGroup(const NextHopKey &nexthop)
```
##### NextHopKey
NextHopKey is an existing structure of the OrchAgent daemon in the SWSS container.  NextHopKey is used by both RouteOrch and NeighOrch to coordinate Next Hop operations.
NextHopKey class is modified to include next hop weight field.
```
struct NextHopKey
{
    IpAddress           ip_address;     // neighbor IP address
    string              alias;          // incoming interface alias
    uint8_t             weight;         // NH weight for NHGs
    ... struct definition abbreviated ...
};
```

### SAI API
This section describes SAI APIs used to support weighted ECMP routes.

Full details about SAI Next Hop Group API and attributes are described here:
https://github.com/opencomputeproject/SAI/blob/master/inc/sainexthopgroup.h
To facilitate weighted ECMP route functionality, the following update was made to the SAI Next Hop Group API:
```
    /**
     * @brief Member weights
     *
     * @type sai_uint32_t
     * @flags CREATE_AND_SET
     * @default 1
     */
    SAI_NEXT_HOP_GROUP_MEMBER_ATTR_WEIGHT,
```

### Configuration and management
There is no Configuration/management change for this feature.

#### CLI Enhancements
There is no CLI change for this feature. Weight information in ECMP is shown in ip route show command.

#### Config DB Enhancements
There is no config DB change.

### Warmboot and Fastboot Design Impact
- Weighted ECMP functionality continues across warm reboot.
- Support for planned system warm restart.
- Support for SWSS docker warm restart.

### Restrictions/Limitations
The routing stack configuration/process sends weight value in netlink messages is not handled in this document.
Example configuartion in FRR might be http://docs.frrouting.org/en/latest/bgp.html#weighted-ecmp-using-bgp-link-bandwidth.

### Testing Requirements/Design
Testing of weighted ECMP can be done either by running routing stack(FRR with BGP), or use ip route command to set weight value in next hop group members.

#### Unit Test cases

#### System Test cases

### Open/Action items

