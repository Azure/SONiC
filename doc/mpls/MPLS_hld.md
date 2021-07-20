# MPLS for SONiC High Level Design Document #

## Table of Content
- [MPLS for SONiC High Level Design Document](#mpls-for-sonic-high-level-design-document)
  - [Table of Content](#table-of-content)
    - [Revision](#revision)
    - [Scope](#scope)
    - [Definitions/Abbreviations](#definitionsabbreviations)
    - [Overview](#overview)
    - [Requirements](#requirements)
      - [Functional Requirements](#functional-requirements)
      - [Configuration and Management Requirements](#configuration-and-management-requirements)
      - [Scalability Requirements](#scalability-requirements)
      - [Warm Boot Requirements](#warm-boot-requirements)
      - [Future Requirements](#future-requirements)
    - [Architecture Design](#architecture-design)
    - [High-Level Design](#high-level-design)
      - [Overview](#overview-1)
      - [Database Changes](#database-changes)
        - [APPL DB](#appl-db)
          - [INTF TABLE](#intf-table)
          - [ROUTE TABLE](#route-table)
          - [LABEL ROUTE TABLE](#label-route-table)
        - [CONFIG DB](#config-db)
          - [INTERFACE](#interface)
          - [PORTCHANNEL INTERFACE](#portchannel-interface)
          - [VLAN INTERFACE](#vlan-interface)
          - [CRM Config](#crm-config)
        - [ASIC DB](#asic-db)
          - [ROUTER INTERFACE](#router-interface-1)
          - [INSEG ENTRY](#inseg-entry)
          - [NEXT HOP](#next-hop-2)
      - [Software Modules](#software-modules)
        - [NetLink](#netlink)
          - [Functions](#functions)
        - [IntfMgr](#intfmgr)
          - [Functions](#functions-1)
        - [FPM Syncd](#fpm-syncd)
          - [Functions](#functions-2)
        - [IntfsOrch](#intfsorch)
          - [Functions](#functions-3)
        - [RouteOrch](#routeorch)
          - [Functions](#functions-4)
        - [NeighOrch](#neighorch)
          - [Functions](#functions-5)
        - [CrmOrch](#crmorch)
          - [Functions](#functions-6)
        - [Label/LabelStack](#label-labelstack)
        - [NextHopKey](#nexthopkey)
        - [Syncd](#syncd)
    - [SAI API](#sai-api)
      - [Router Interface](#router-interface-1)
      - [MPLS](#mpls)
      - [Next Hop](#next-hop-2)
    - [Configuration and management](#configuration-and-management)
      - [CLI Enhancements](#cli-enhancements)
      - [Config DB Enhancements](#config-db-enhancements)
      - [YANG Model Enhancements](#yang-model-enhancements)
        - [SONiC Interface](#sonic-interface)
        - [SONiC VLAN](#sonic-vlan)
        - [SONiC PortChannel](#sonic-portchannel)
        - [SONiC CRM](#sonic-crm)
   - [Warmboot and Fastboot Design Impact](#warmboot-and-fastboot-design-impact)
    - [Restrictions/Limitations](#restrictionslimitations)
    - [Testing Requirements/Design](#testing-requirementsdesign)
      - [Unit Test cases](#unit-test-cases)
      - [System Test cases](#system-test-cases)
    - [Open/Action items - if any](#openaction-items---if-any)

### Revision
|  Rev  |  Date       |  Author  | Change Description |
| :---: | :---------: | :------: | :----------------: |
|  0.1  | Jan-10-2021 | A Pokora | Initial version    |
|  0.2  | Jan-19-2021 | A Pokora | Updates from MPLS sub-community review |
|  0.3  | Jun-14-2021 | A Pokora | Updates from MPLS sub-community code-review |

### Scope

This document provides general information about the initial support for MPLS in SONiC infrastructure.  The focus of this initial MPLS support is to expand existing SONiC infrastructure for IPv4/IPv6 routing to include equivalent MPLS functionality.  The expected use case for this initial MPLS support is static LSP routing.

### Definitions/Abbreviations
| Abbreviation | Description                           |
| :----------: | :-----------------------------------: |
| CRM          | Critical Resource Monitoring          |
| cRPD         | Containerized Routing Protocol Daemon |
| LSP          | Label-Switched Path                   |
| MPLS         | Multi-Protocol Label Switching        |

### Overview
This document provides general information about the initial support for MPLS in SONiC infrastructure.

### Requirements
This section describes the requirements for the initial support for MPLS in SONiC infrastructure.

#### Functional requirements
- Support for MPLS enable/disable per Router Interface.
- Support for MPLS Push, Pop, and Swap label operations, including MPLS implicit-null and explicit-null behavior.
- Support for bulk MPLS in-segment entry SAI programming.
- Support for MPLS type next-hop SAI programming.
- Support in CRM for MPLS in-segment entries and MPLS next-hops accounting.
- Support for VS platform SAI for test purposes.

#### Configuration and Management requirements
- SONiC CLI support for configuring MPLS enable/disable per Router Interface.
- SONiC CLI support for displaying MPLS state per Router Interface.
- SONiC CLI support for configuring CRM thresholds for MPLS in-segment entries and MPLS type next-hops.
- SONiC CLI support for displaying CRM thresholds and accouting for MPLS in-segment entries and MPLS type next-hops.

#### Scalability Requirements
- Up to max ASIC capable MPLS in-segment entries are supported.
- Error is logged in syslog for all attempted MPLS routes after max limit is reached.
- CRM notification upon reaching configurable scaling thresholds.

#### Warm Boot Requirements
- MPLS functionality continues across warm reboot.
- Support for planned system warm restart.
- Support for SWSS docker warm restart.

#### Future Requirements
- SONiC CLI support for MPLS operational commands.
- FRR Zebra FPM support for MPLS in-segment entries and MPLS next-hops.
- Support for VRFs

### Architecture Design
For MPLS, SONiC SwSS infrastructure Route and Next Hop support is extended to include optional MPLS label stack in addition to the existing IPv4/IPv6 address information.

### High-Level Design

#### Overview
![Overview diagram](images/MPLS_overview_diagram.png "Overview of MPLS components")

**Figure 1: Overview of the data flow and related components of MPLS**

#### Database Changes
This section describes the modifications to SONiC Databases to support MPLS.

##### APPL DB

###### INTF TABLE
The existing INTF_TABLE in the APPL_DB is enhanced to accept a new "mpls" enable/disable attribute.

```
INTF_TABLE|{{interface_name}}
    "mpls":{{enable|disable}} (OPTIONAL)

; Defines schema for MPLS configuration attribute
key          = INTERFACE:ifname      ; Interface name
; field      = value
mpls         = "enable" / "disable"  ; Enable/disable MPLS function. Default "disable"
```

###### ROUTE TABLE
The existing ROUTE_TABLE for IPv4/IPv6 prefix routes in the APPL_DB is enhanced to accept an optional "mpls_nh" attribute that is applicable when a MPLS push operation is configured.  The format of the "mpls_nh" attribute string for IPv4/IPv6 prefix routes is: "push+label0/.../labelN".

For IP forward-only next-hops, the "mpls_nh" attribute is not applicable.  If the IPv4/IPv6 prefix route is associated with a single IP forward-only next-hop or a next-hop group consisting only of these hext-hops, then the "mpls_nh" attribute will not be present.  If the IPv4/IPv6 prefix route is associated with a next-hop group with a mix of MPLS push and IP forward-only next-hops, then each IP forward-only nexthop will be represented by "na" in the "mpls_nh" attribute.

For all next-hop types, the formats of the "nexthop" and "ifname" attributes are unchanged from previous releases.

```
"ROUTE_TABLE":{{prefix}}
    "nexthop":{{nexthop_list}}
    "ifname":{{ifname_list}}
    "mpls_nh":{{mpls_nh_list}}

; Defines schema for IPv4/IPv6 route table attributes
key          = ROUTE_TABLE:prefix    ; IPv4/IPv6 prefix
; field      = value
nexthop      = STRING                ; Comma-separated list of IP gateways.
ifname       = STRING                ; Comma-separated list of interfaces.
mpls_nh      = STRING                ; Comma-separated list of MPLS next-hop info.
```

###### LABEL ROUTE TABLE
A new LABEL_ROUTE_TABLE is introduced to the APPL_DB for MPLS in-segment entries.  The LABEL_ROUTE_TABLE uses the ingress MPLS label as its lookup key, instead of the IP prefix used by the ROUTE_TABLE.
The LABEL_ROUTE_TABLE accepts the same attributes as ROUTE_TABLE:
- A "nexthop" formatted-string attribute containing a list of IP gateways.
- A "ifname" attribute containing a list of interfaces.
- A "mpls_nh" attribute containing a list MPLS next-hop info.

For MPLS in-segment routes, the "mpls_nh" attribute is applicable when a MPLS swap operation is configured.  The format of the "mpls_nh" attribute for MPLS in-segment routes is: "swap+label0/../labelN".

For MPLS pop and IP forward-only operations, the "mpls_nh" attribute is not applicable.  If the MPLS in-segment entry is associated with a single MPLS pop or IP forward-only next-hop or a next-hop group consisting only of htese next-hops, then the "mpls_nh" attribute will not be present.  If the MPLS in-segment etry is associated with a next-hop group with a mix of MPLS swap and MPLS pop/IP forward-only next-hops, then each MPLS pop/IP forward-only next-hop will be represented by "na" in the "mpls_nh" attribute.

The LABEL_ROUTE_TABLE will contain an additional "mpls_pop" attribute for each MPLS in-segment entry.  The value of "mpls_pop" will be "0" if the ingress MPLS label is to be retained (ie, IP forward-only next-hop).  The value of "mpls_pop" will be "1" if the ingress MPLS label is to be removed (ie, MPLS pop or MPLS swap next-hop).

For all next-hop types, the formats of the "nexthop" and "ifname" attributes are unchanged from previous releases.

For MPLS "implicit-null" operations, the "mpls_nh" attribute is not present and the expected  "mpls_pop" attribute value is "1" (ie, it is a MPLS pop next-hop)

For MPLS "explicit-null" operations, the expected "mpls_nh" attribute value is "swap+0" and the expected "mpls_pop" attribute value is "1" (ie, it is a special case of a MPLS swap next-hop).

```
"LABEL_ROUTE_TABLE":{{mpls_label}}
    "nexthop":{{nexthop_list}}
    "ifname":{{ifname_list}}
    "mpls_nh":{{mpls_nh_list}}
    "mpls_pop":{{mpls_pop}}

; Defines schema for MPLS label route table attributes
key           = LABEL_ROUTE_TABLE:mpls_label ; MPLS label
; field       = value
nexthop       = STRING           ; Comma-separated list of nexthops.
ifname        = STRING           ; Comma-separated list of interfaces.
mpls_nh       = STRING           ; Comma-separated list of MPLS NH info.
mpls_pop      = STRING           ; Number of ingress MPLS labels to POP
```

##### CONFIG DB

###### INTERFACE
The existing INTERFACE table is enhanced to accept a new "mpls" enable/disable attribute.

```
INTERFACE|{{ifname}}
    "mpls":{{enable|disable}} (OPTIONAL)

; Defines schema for MPLS configuration attribute
key          = INTERFACE:ifname    ; Interface name
; value annotations
ifname       = 1*64VCHAR           ; name of the Interface
; field      = value
mpls         = "enable"/"disable"  ; Enable/disable MPLS function. Default "disable"
```

###### PORTCHANNEL INTERFACE
The existing PORTCHANNEL_INTERFACE table is enhanced to accept a new "mpls" enable/disable attribute.

```
PORTCHANNEL_INTERFACE|{{ifname}}
    "mpls":{{enable|disable}} (OPTIONAL)

; Defines schema for MPLS configuration attributes
key         = PORTCHANNEL_INTERFACE:ifname  ; Port Channel Interface name
;value annotations
ifname      = 1*64VCHAR                     ; name of the Interface (Port Channel)
; field     = value
mpls        = "enable"/"disable"            ; Enable/disable MPLS function. Default "disable"
```

###### VLAN INTERFACE
The existing VLAN_INTERFACE table is enhanced to accept a new "mpls" enable/disable attribute.

```
VLAN_INTERFACE|{{ifname}}
    "mpls":{{enable|disable}} (OPTIONAL)

; Defines schema for MPLS configuration attributes
key         = VLAN_INTERFACE:ifname          ; VLAN Interface name
;value annotations
ifname      = 1*64VCHAR                      ; name of the Interface (VLAN)
; field     = value
mpls        = "enable"/"disable"             ; Enable/disable MPLS function. Default "disable"
```

###### CRM Config
The existing CRM Config stanza is enhanced to include new MPLS in-segment entry and MPLS next-hop attributes.  These attributes parallel existing CRM configuration for other resource types (eg, IPv4/IPv6 routes and next-hops).

```
CRM
  Config
    "mpls_inseg_threshold_type":{{percentage|used|free}} (OPTIONAL)
    "mpls_inseg_high_threshold":{{UINT32}} (OPTIONAL)
    "mpls_inseg_low_threshold":{{UINT32}} (OPTIONAL)

    "mpls_nexthop_threshold_type":{{percentage|used|free}} (OPTIONAL)
    "mpls_nexthop_high_threshold":{{UINT32}} (OPTIONAL)
    "mpls_nexthop_low_threshold":{{UINT32}} (OPTIONAL)

; Defines schema for CRM MPLS in-segment entry and MPLS next-hop configuration attributes
; field                    = value
mpls_inseg_threshold_type  = "percentage"/"used"/"free" ; Threshold type. Default "percentage"
mpls_inseg_high_threshold  = UINT32             ; High threshold. Default value = 85
mpls_inseg_low_threshold   = UINT32             ; Low threshold. Default value = 70

mpls_nexthop_threshold_type  = "percentage"/"used"/"free" ; Threshold type. Default "percentage"
mpls_nexthop_high_threshold  = UINT32             ; High threshold. Default value = 85
mpls_nexthop_low_threshold   = UINT32             ; Low threshold. Default value = 70
```

#### ASIC DB

##### ROUTER INTERFACE
Support for a new attribute is introduced to the ASIC_DB for the existing ROUTER_INTERFACE object type:

##### INSEG ENTRY
Support for a new object type INSEG_ENTRY is introduced to the ASIC_DB:

##### NEXT HOP
Support for new attributes are introduced to the ASIC_DB for the existing NEXT_HOP object type:

#### Software Modules
This section describes modifications to SONiC infrastructure software modules to support MPLS.

##### NetLink Library
The Netlink library (libnl3) is an existing open source library imported by SONiC to parse and format Netlink messages.
Modifications to the existing NetLink library MPLS implementation were needed to support MPLS attributes.
###### Functions
New Netlink message next-hop accessors were added to retrieve attributes in a nested MPLS encapsulation (RTA_ENCAP_TYPE of LWTUNNEL_IPTUNNEL_MPLS) stanza.  The following accessors retrieve the value for the attributes of MPLS next-hop destination (MPLS_IPTUNNEL_DST) and TTL (MPLS_IPTUNNEL_TTL):
```
  /* Accessor to retrieve MPLS destination */
  extern struct nl_addr *	rtnl_route_nh_get_encap_mpls_dst(struct rtnl_nexthop *);
  /* Accessor to retrieve MPLS TTL */
  extern uint8_t		rtnl_route_nh_get_encap_mpls_ttl(struct rtnl_nexthop *);
```

##### IntfMgr
IntfMgr is an existing daemon in SWSS container that monitors operations in CONFIG_DB on INTERFACE, PORTCHANNEL_INTERFACE, and VLAN_INTERFACE tables.

For MPLS, IntfMgr is modified to additionally process the "mpls" enable/disble attribute from the CONFIG_DB and propagate this attribute to APPL_DB.
###### Functions
The following are functions for IntfMgr:
```
  /* MPLS enable/disable per Interface */
  bool IntfMgr::setIntfMpls(const std::string &alias, const std::string &mpls);
```
This function sets the Linux kernel variable net.mpls.interface.\<intf-name\> to enable/disable MPLS on the specified interface.

##### FPM Syncd
FPM Syncd is an existing daemon in BGP container that monitors NetLink route messages (RTM_NEWROUTE and RTM_DELROUTE) from the SONiC routing stack FPM socket for route and next-hop information.

New support has been added to FPM Syncd for MPLS to process MPLS related route and next-hop information in the received NetLink messages and propagate this information to the APPL_DB.
###### Functions
The following are new functions for fpmsyncd:
```
  /* Handler for rtnl messages with AF_MPLS route */
  void RouteSync::onLabelRouteMsg(int nlmsg_type, struct nl_object *obj);
  /* Handler for rtnl messages with IP and/or MPLS next-hops */
  void RouteSync::getNextHopList(struct rtnl_route *route_obj, string& gw_list,
                                 string& mpls_list, string& intf_list);

```

##### IntfsOrch
IntfsOrch is an existing component of the OrchAgent daemon in the SWSS container.  IntfsOrch monitors operations on Interface related tables in APPL_DB and converts those operations into SAI commands to manage the Router Interface object.

For MPLS, IntfsOrch has been extended to detect the new per-RIF "mpls" enable/disable attribute in the APPL_DB and propagate this configuration to the ASIC_DB via SAI_ROUTER_INTERFACE_ATTR_ADMIN_MPLS_STATE.  This MPLS behavior parallels the existing IntfsOrch behavior of SAI_ROUTER_INTERFACE_ATTR_ADMIN_V4_STATE and SAI_ROUTER_INTERFACE_ATTR_ADMIN_V6_STATE for IPv4/IPv6.
###### Functions
The following are new functions for IntfsOrch:
```
  /* Handler to enable/disable MPLS per Interface */
  bool IntfsOrch::setRouterIntfMpls(const Port& port)
```

##### RouteOrch
RouteOrch is an existing component of the OrchAgent daemon in the SWSS container.  RouteOrch monitors operations on Route related tables in APPL_DB and converts those operations in SAI commands to manage IPv4/IPv6 route and MPLS in-segment entries.  Additionally RouteOrch coordinates next-hop object operations with NeighOrch and converts operations into SAI commands to manage next-hop group objects.

For MPLS, RouteOrch was modified to monitor updates to the new APPL_DB LABEL_ROUTE_TABLE.  RouteOrch translates all updates to LABEL_ROUTE_TABLE to equivalent SAI requests for SAI MPLS inseg API.
NextHop processing for updates from both the new LABEL_ROUTE_TABLE and the existing ROUTE_TABLE has been extended to detect possible MPLS attributes and propagate this additional information to NeighOrch for SAI handling.
###### Functions
The following are new functions for RouteOrch:
```
  /* Consumer handler for all events in APPL_DB LABEL_ROUTE_TABLE */
  void RouteOrch::doLabelTask(Consumer& consumer);
  /* Handler to process new MPLS route from LABEL_ROUTE_TABLE */
  bool RouteOrch::addLabelRoute(LabelRouteBulkContext& ctx, const NextHopGroupKey&);
  /* Handler to process MPLS route removal from LABEL_ROUTE_TABLE */
  bool RouteOrch::removeLabelRoute(LabelRouteBulkContext& ctx);
```

##### NeighOrch
NeighOrch is an existing component of the OrchAgent daemon in the SWSS container.  NeighOrch monitors operations on Neighbor related tables in APPL_DB.  Additionally NeighOrch coordinates next-hop operations with RouteOrch and converts operations into SAI commands to manage next-hop objects.

For MPLS, NeighOrch has been extended to send create/remove SAI requests for MPLS next-hop objects (ie, next-hop objects of type SAI_NEXT_HOP_TYPE_MPLS) when associated neighbor objects are created/removed. This MPLS next-hop behavior parallels the existing IPv4/IPv6 next-hop behavior in NeighOrch.
###### Functions
Existing functions from NeighOrch are updated to include NextHopKey parameter instead of IpAddress and visibility is raised to public for RouteOrch accessibility.
```
    bool addNextHop(const NextHopKey&);
    bool removeNextHop(const NextHopKey&);
```

##### CrmOrch
CrmOrch is an existing component of the OrchAgent daemon in the SWSS container.  CrmOrch monitors resource usage in the SONiC system and triggers alarms when configurable thresholds are reached.

For MPLS, CrmOrch has been extended to monitor the number of MPLS in-segment entries and MPLS next-hops against the platform-specific number of entries available.  To facilitate this, new CRM resource types of CRM_MPLS_INSEG and CRM_MPLS_NEXTHOP have been added to CrmOrch.
CRM_MPLS_INSEG has been mapped to the existing SAI object type SAI_OBJECT_TYPE_INSEG_ENTRY for querying via sai_object_type_get_availability().  This MPLS behavior parallels the existing CRM_ROUTE_IPV4 behavior with SAI_SWITCH_ATTR_AVAILABLE_IPV4_ROUTE_ENTRY and CRM_IPV6_ROUTE behavior with SAI_SWITCH_ATTR_AVAILABLE_IPV6_ROUTE_ENTRY.
CRM_MPLS_NEXTHOP has been mapped to existing SAI object type SAI_OBJECT_TYPE_NEXT_HOP with SAI_NEXT_HOP_ATTR_TYPE of SAI_NEXT_HOP_TYPE_MPLS for querying via sai_object_type_get_availability().  This MPLS behavior parallels the existins CRM_NEXTHOP_IPV4 behavior with SAI_SWITCH_ATTR_AVAILABLE_IPV4_NEXT_HOPS and CRM_NEXTHOP_IPV6 behavior with SAI_SWITCH_AVAILABLE_IPV6_NEXT_HOPS.
###### Functions
No new functions were required for CrmOrch MPLS in-segment entry and next-hop support.

##### Label/LabelStack
Label and LabelStack are new type utilities of the OrchAgent daemon in the SWSS container.
These types are introduced to represent the MPLS label or label stack when found in an in-segment entry or next hop.
```
typedef uint32_t Label;
class LabelStack
{
    ... class definition abbreviated ...
private:
    std::set<Label> m_labelstack;
};
```
##### NextHopKey
NextHopKey is an existing utility of the OrchAgent daemon in the SWSS container.  NextHopKey is used by both RouteOrch and NeighOrch to coordinate next-hop operations.

For MPLS, the NextHopKey struct is modified to include a sai_next_hop_type_t of SAI_NEXT_HOP_TYPE_MPLS, LabelStack field and sai_outseg_type_t field.
```
struct NextHopKey
{
    sai_next_hop_type_t nh_type;        // next-hop type
    IpAddress           ip_address;     // neighbor IP address
    string              alias;          // incoming interface alias
    LabelStack          label_stack;    // MPLS label stack
    sai_outseg_type_t   outseg_type;    // MPLS out-segment type (swap | push)
    ... struct definition abbreviated ...
};
```
The LabelStack and sai_outseg_type_t fields in NextHopKey are not applicable for NextHopKey with sai_next_hop_type_t value other than SAI_NEXT_HOP_TYPE_MPLS.

##### Syncd
Syncd is an existing daemon of the Syncd container which handles all events driven by the ASIC_DB.
For MPLS, modifications were made to fully support sairedis handling of the existing SAI MPLS API key: sai_inseg_entry_t.
```
/**
 * @brief In segment entry
 */
typedef struct _sai_inseg_entry_t
{
    /**
     * @brief Switch ID
     *
     * @objects SAI_OBJECT_TYPE_SWITCH
     */
    sai_object_id_t switch_id;

    /**
     * @brief MPLS label
     */
    sai_label_id_t label;

} sai_inseg_entry_t;
```

### SAI API
This section describes SAI APIs used and enhanced to support MPLS.

#### Router Interface
Full details about current SAI Router Interface API and attributes are described here:
https://github.com/opencomputeproject/SAI/blob/master/inc/sairouterinterface.h

To facilitate MPLS functionality, the following update was made to the SAI Router Interface API:
```
    /**
     * @brief Admin MPLS state
     *
     * @type bool
     * @flags CREATE_AND_SET
     * @default false
     */
    SAI_ROUTER_INTERFACE_ATTR_ADMIN_MPLS_STATE,
```

#### MPLS
Full details about current SAI MPLS API and attributes are described here:
https://github.com/opencomputeproject/SAI/blob/master/inc/saimpls.h

The entirety of the existing SAI MPLS API definition is now introduced to SONiC orchagent to facilitate MPLS route functionality.

Additionally, the following has been added to the SAI MPLS API definition for to facilitate bulk MPLS route operations for capable platforms:
```
/**
 * @brief Bulk create In Segment entry
 *
 * @param[in] object_count Number of objects to create
 * @param[in] inseg_entry List of object to create
 * @param[in] attr_count List of attr_count. Caller passes the number
 *    of attribute for each object to create.
 * @param[in] attr_list List of attributes for every object.
 * @param[in] mode Bulk operation error handling mode.
 * @param[out] object_statuses List of status for every object. Caller needs to
 * allocate the buffer
 *
 * @return #SAI_STATUS_SUCCESS on success when all objects are created or
 * #SAI_STATUS_FAILURE when any of the objects fails to create. When there is
 * failure, Caller is expected to go through the list of returned statuses to
 * find out which fails and which succeeds.
 */
typedef sai_status_t (*sai_bulk_create_inseg_entry_fn)(
        _In_ uint32_t object_count,
        _In_ const sai_inseg_entry_t *inseg_entry,
        _In_ const uint32_t *attr_count,
        _In_ const sai_attribute_t **attr_list,
        _In_ sai_bulk_op_error_mode_t mode,
        _Out_ sai_status_t *object_statuses);

/**
 * @brief Bulk remove In Segment entry
 *
 * @param[in] object_count Number of objects to remove
 * @param[in] inseg_entry List of objects to remove
 * @param[in] mode Bulk operation error handling mode.
 * @param[out] object_statuses List of status for every object. Caller needs to
 * allocate the buffer
 *
 * @return #SAI_STATUS_SUCCESS on success when all objects are removed or
 * #SAI_STATUS_FAILURE when any of the objects fails to remove. When there is
 * failure, Caller is expected to go through the list of returned statuses to
 * find out which fails and which succeeds.
 */
typedef sai_status_t (*sai_bulk_remove_inseg_entry_fn)(
        _In_ uint32_t object_count,
        _In_ const sai_inseg_entry_t *inseg_entry,
        _In_ sai_bulk_op_error_mode_t mode,
        _Out_ sai_status_t *object_statuses);

/**
 * @brief Bulk set attribute on In Segment entry
 *
 * @param[in] object_count Number of objects to set attribute
 * @param[in] inseg_entry List of objects to set attribute
 * @param[in] attr_list List of attributes to set on objects, one attribute per object
 * @param[in] mode Bulk operation error handling mode.
 * @param[out] object_statuses List of status for every object. Caller needs to
 * allocate the buffer
 *
 * @return #SAI_STATUS_SUCCESS on success when all objects are removed or
 * #SAI_STATUS_FAILURE when any of the objects fails to remove. When there is
 * failure, Caller is expected to go through the list of returned statuses to
 * find out which fails and which succeeds.
 */
typedef sai_status_t (*sai_bulk_set_inseg_entry_attribute_fn)(
        _In_ uint32_t object_count,
        _In_ const sai_inseg_entry_t *inseg_entry,
        _In_ const sai_attribute_t *attr_list,
        _In_ sai_bulk_op_error_mode_t mode,
        _Out_ sai_status_t *object_statuses);

/**
 * @brief Bulk get attribute on In Segment entry
 *
 * @param[in] object_count Number of objects to set attribute
 * @param[in] inseg_entry List of objects to set attribute
 * @param[in] attr_count List of attr_count. Caller passes the number
 *    of attribute for each object to get
 * @param[inout] attr_list List of attributes to set on objects, one attribute per object
 * @param[in] mode Bulk operation error handling mode
 * @param[out] object_statuses List of status for every object. Caller needs to
 * allocate the buffer
 *
 * @return #SAI_STATUS_SUCCESS on success when all objects are removed or
 * #SAI_STATUS_FAILURE when any of the objects fails to remove. When there is
 * failure, Caller is expected to go through the list of returned statuses to
 * find out which fails and which succeeds.
 */
typedef sai_status_t (*sai_bulk_get_inseg_entry_attribute_fn)(
        _In_ uint32_t object_count,
        _In_ const sai_inseg_entry_t *inseg_entry,
        _In_ const uint32_t *attr_count,
        _Inout_ sai_attribute_t **attr_list,
        _In_ sai_bulk_op_error_mode_t mode,
        _Out_ sai_status_t *object_statuses);
```
The existing sai_mpls_api_t structure has been expanded to accommodate the new MPLS bulking APIs:
```
/**
 * @brief MPLS methods table retrieved with sai_api_query()
 */
typedef struct _sai_mpls_api_t
{
     sai_create_inseg_entry_fn                      create_inseg_entry;
     sai_remove_inseg_entry_fn                      remove_inseg_entry;
     sai_set_inseg_entry_attribute_fn               set_inseg_entry_attribute;
     sai_get_inseg_entry_attribute_fn               get_inseg_entry_attribute;

+    sai_bulk_create_inseg_entry_fn                 create_inseg_entries;
+    sai_bulk_remove_inseg_entry_fn                 remove_inseg_entries;
+    sai_bulk_set_inseg_entry_attribute_fn          set_inseg_entries_attribute;
+    sai_bulk_get_inseg_entry_attribute_fn          get_inseg_entries_attribute;

} sai_mpls_api_t;
```

#### Next Hop
Full details about SAI Next Hop API and attributes are described here:
https://github.com/opencomputeproject/SAI/blob/master/inc/sainexthop.h

No modifications were made to the current SAI Next Hop API definition.

The following existing attributes are now introduced to SONiC orchagent to facilitate MPLS NextHop functionality:
```
    /**
     * @brief Push label
     *
     * @type sai_u32_list_t
     * @flags MANDATORY_ON_CREATE | CREATE_ONLY
     * @condition SAI_NEXT_HOP_ATTR_TYPE == SAI_NEXT_HOP_TYPE_MPLS
     */
    SAI_NEXT_HOP_ATTR_LABELSTACK,

    /**
     * @brief MPLS Outsegment type
     *
     * @type sai_outseg_type_t
     * @flags CREATE_AND_SET
     * @default SAI_OUTSEG_TYPE_SWAP
     * @validonly SAI_NEXT_HOP_ATTR_TYPE == SAI_NEXT_HOP_TYPE_MPLS
     */
    SAI_NEXT_HOP_ATTR_OUTSEG_TYPE,

```

### Configuration and management
This section should have sub-sections for all types of configuration and management related design. Example sub-sections for "CLI" and "Config DB" are given below. Sub-sections related to data models (YANG, REST, gNMI, etc.,) should be added as required.

#### CLI Enhancements

##### Configuration CLI Commands
A new SONiC CLI command is introduced to configure interfaces for MPLS.

    # Enable/disable MPLS per INTERFACE/PORTCHANNEL_INTERFACE/VLAN_INTERFACE.
    config interface mpls add|remove <intf-name>

##### Show CLI Commands
A new SONiC CLI command is introduced to display the current MPLS configuraiton for interfaces.

    # Show MPLS state per INTERFACE/PORT_CHANNEL_INTERFACE/VLAN_INTERFACE
    show interface mpls [<intf-name>]

Example output from the above command:
```
  admin@sonic:~$ show interfaces mpls
  Interface    MPLS State
  -----------  ------------
  Ethernet0    disable
  Ethernet4    enable
  Ethernet8    enable
  Ethernet12   disable
  Ethernet16   disable
  Ethernet20   disable

  admin@sonic:~$ show interfaces mpls Ethernet4
  Interface    MPLS State
  -----------  ------------
  Ethernet4    enable

```

#### Config DB Enhancements
For details please refer to [CONFIG_DB](#config-db)

#### YANG Model Enhancements

##### SONiC Interface
The existing sonic-interface.yang model is enhanced to support a new "mpls" enable/disable attribute.

```
  container sonic-interface {
    container INTERFACE {
      list INTERFACE_LIST {
+       leaf mpls {
+         description "Enable/disable MPLS routing for the interface";
+         type string {
+           pattern "enable|disable";
+         }
+       }
      }
    }
  }
```

##### SONiC VLAN
The existing sonic-vlan.yang model is enhanced to support a new "mpls" enable/disable attribute.

```
  container sonic-vlan {
    container VLAN_INTERFACE {
      list VLAN_INTERFACE_LIST {
+       leaf mpls {
+         description "Enable/disable MPLS routing for the vlan interface";
+         type string {
+           pattern "enable|disable";
+         }
+       }
      }
    }
  }
```

##### SONiC PortChannel
The existing sonic-portchannel.yang model is enhanced to support a new "mpls" enable/disable attribute.

```
  container sonic-portchannel {
    container PORTCHANNEL_INTERFACE {
      list PORTCHANNEL_INTERFACE_LIST {
+       leaf mpls {
+         description "Enable/disable MPLS routing for the portchannel interface";
+         type string {
+           pattern "enable|disable";
+         }
+       }
      }
    }
  }
```

##### SONiC CRM
The existing sonic-crm.yang model is enhanced to support the new MPLS in-segment entry and MPLS next-hop CRM thresholds.

```
container sonic-crm {
  container CRM {
    container Config {
+     leaf mpls_inseg_threshold_type {
+       must "(((current()='PERCENTAGE' or current()='percentage') and
+             ../mpls_inseg_high_threshold<100 and
+             ../mpls_inseg_low_threshold<100) or
+             (current()!='PERCENTAGE' and current()!='percentage'))";
+       type stypes:crm_threshold_type;
+     }

+     leaf mpls_inseg_high_threshold {
+       must "(current() > ../mpls_inseg_low_threshold)"
+       {
+         error-message "high_threshold should be more than low_threshold";
+       }
+       type threshold;
+     }

+     leaf mpls_inseg_low_threshold {
+       type threshold;
+     }

+     leaf mpls_nexthop_threshold_type {
+       must "(((current()='PERCENTAGE' or current()='percentage') and
+             ../mpls_nexthop_high_threshold<100 and
+             ../mpls_nexthop_low_threshold<100) or
+             (current()!='PERCENTAGE' and current()!='percentage'))";
+       type stypes:crm_threshold_type;
+     }

+     leaf mpls_nexthop_high_threshold {
+       must "(current() > ../mpls_nexthop_low_threshold)"
+       {
+         error-message "high_threshold should be more than low_threshold";
+       }
+       type threshold;
+     }

+     leaf mpls_nexthop_low_threshold {
+       type threshold;
+     }

    }
  }
}
```

### Warmboot and Fastboot Design Impact
This SONiC infrastructure support for MPLS is an enhancement of existing IPv4/IPv6 routing infrastructure which allows it to make use of existing warmboot and fastboot handling.  For this reason, MPLS design will not affect warmboot or fastboot design.

### Restrictions/Limitations
- No support for programming default MPLS routes for IPv4/IPv6 Explicit NULL from SONiC infrastructure. These default MPLS routes must be programmed/handled by vendor SAI implementation.  This limitation is due to lack of support for Route Table as valid Next Hop entity in SAI MPLS definition.
- Outermost ingress MPLS label will always be popped.  This limitation is due to implicit pop in Linux/Netlink implementation.

### Testing Requirements/Design
An external routing controller is used to set up static LSP route for push/pop/swap operation on MPLS traffic and verify traffic is passing.

#### Unit Test cases
- Add/remove IPv4 route entry with associated MPLS push next-hop.
- Add/remove MPLS in-segment entry with associated MPLS swap next-hop.
- Add/remove MPLS in-segment entry with associated MPLS implicit-null (pop) next-hop.
- Add/remove MPLS in-segment entry with associated MPLS explicit-null (swap) next-hop.
- Add/remove IPv4 route entry with associated next-hop group of two MPLS push next-hops.
- Add/remove MPLS in-segment entry with associated next-hop group of two MPLS swap next-hops.
- Add/remove MPLS in-segment entry with associated next-hop group of two MPLS implicit-null (pop) next-hops.
- Add/remove IPv4 route entry with associated next-hop group of one MPLS push and one IPv4 forward-only nexthop.
- Add/remove MPLS in-segment entry with associated next-hop group of one MPLS swap and one MPLS implicit-null (pop) nexthop.
- Add/remove MPLS in-segment entry with unresolved MPLS swap next-hop.  Verify both in-segment and next-hop are not programmed until next-hop is resolved.

#### System Test cases
- Add IPv4 route entry with associated MPLS push next-hop.  Verify ingress IP traffic and egress MPLS traffic with correct MPLS label.
- Add MPLS in-segment entry with associated MPLS implicit-null (pop) next-hop.  Verify ingress MPLS traffic and egress IP traffic.
- Add MPLS in-segment entry with associated MPLS swap next-hop.  Verify ingress MPLS traffic and egress MPLS traffic with correct MPLS label.

### Open/Action items - if any
None
