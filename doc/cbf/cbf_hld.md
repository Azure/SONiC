
# Class Based Forwarding Enhancement
#### Rev 0.1

# Table of Contents
  * [Revision](#revision)
  * [About This Manual](#about-this-manual)
  * [1. Introduction](#1-introduction)
  * [2. Requirements Overview](#2-requirement-overview)
      * [2.1 Functional Requirements](#21-functional-requirements)
      * [2.2 Configuration and Management Requirements](#22-configuration-and-management-requirements)
      * [2.3 Scalability Requirements](#23-scalability-requirements)
      * [2.4 Warm Boot Requirements](#24-warm-boot-requirements)
      * [2.4 Restrictions](#25-restrictions)
  * [3. Design](#3-design)
      * [3.1 Overview](#31-overview)
      * [3.2 DB Changes](#32-db-changes)
          * [3.2.1 APPL DB](#321-appl-db)
          * [3.2.2 CONFIG DB](#322-config-db)
      * [3.3 Switch State Service Design](#33-switch-state-service-design)
          * [3.3.1 Orchestration Agent](#331-orchestration-agent)
      * [3.4 sairedis](#34-sairedis)
      * [3.5 SAI](#35-sai)
      * [3.6 CLI](#36-cli)
  * [4. Warm Boot Support](#4-warm-boot-support)
      * [4.1 Warm Upgrade](#31-warm-upgrade)
  * [5. Unit Test](#5-unit-test)


# Revision
| Rev |     Date    |       Author            | Change Description                         |
|:---:|:-----------:|:-----------------------:|--------------------------------------------|
| 0.1 | 03/06/2021  |     Alexandru Banu      | Initial version                            |


# About this Manual
This document provides general information about Class Based Forwarding which allows traffic to be steered through the network by policy, adding a layer of traffic engineering based on a Forwarding Class value which allows custom paths to be configured for a destination based on this value.

Along this document the following abbreviations might be used:

FC - Forwarding Class
CBF - Class Based Forwarding
NHG - Next Hop Group

# 1 Introduction
Class Based Forwarding allows the routed traffic according to the IP/MPLS decision rules to be forwarded on different paths for the same destination depending on the Forwarding Class (different from the Traffic Class), which is determined by a mapping from the DSCP/EXP value of the packet to the Forwarding Class value. A packet coming in with a DSCP/EXP value of X will receive a Forwarding Class (FC) value of Y according to the mapping table provided at the start-of-day. This packet will then be routed, as mentioned earlier, using the traditional IP/MPLS lookup. If the chosen route uses Class Based Forwarding, the next hop will be chosen based on the Forwarding Class value. You can find a flow diagram describing this below:

```
Packet is received with     A lookup is performed      FC value X is      IP routing decision     Routing lookup returns      The next hop group Z is       Packet is forwarded
DSCP/EXP value of W for --> in the DSCP/EXP to FC --> assigned to the --> lookup is performed --> next hop group Y, which --> selected from the members --> via group Z to the
   destination D               map table for W            packet           for destination D          is a CBF group            of Y based on the FC           destination D
                                                                                                                                       value X
```
This feature enables opeartors, among other things, to send the important (foreground) traffic through the shortest path, while sending the background traffic through longer paths to still give it some bandwidth instead of using QoS queues which may block background traffic from getting bandwitdh.

These new class based next hop groups are allowed thanks to the changes in https://github.com/opencomputeproject/SAI/pull/1193, which allow a next hop group object to also have other next hop group objects as members of the group along with the next hop objects. The way such a next hop group works is that a packet which has a Forwarding Class value of X will be matched against an appropriate member of this group, selected based on the Forwarding Class value thanks to the "class_map" property of the group. As an example, given the CBF group with members Nhg1, Nhg2 and Nhg3 and a class map of FC 0 -> Nhg1, FC 1 -> Nhg2 and FC 3 -> Nhg3, a packet which has an FC value of 0 will be forwarded using Nhg1. Note that multiple FC values can point to the same member, but a single FC value can't be mapped to more than one member.

In order to support this mapping, 2 new mapping tables will be added to the CONFIG_DB for the DSCP/EXP to FC mapping and a new CLASS_BASED_NEXT_HOP_GROUP table will be added to APPL_DB to support the new FC-aware next hop groups.

# 2 Requirement Overview
## 2.1 Functional Requirements

Allow traffic to be forwarded through the network based on their DSCP/EXP values following these rules:
- If a packet is not matched against an FC value and the route for its destination does not reference a CBF NHG, the packet will use the route's NH
- If a packet is not matched against an FC value and the route for its destination references a CBF NHG, the packet will be dropped
- If a packet is matched against an FC value and the route for its destination does not reference a CBF NHG, the packet will use the route's NH
- If a packet is matched against an FC value and the route for its destination references a CBF NHG which maps the packet's FC value, the packet will use the mapped NHG
- If a packet is matched against an FC value and the route for its destination references a CBF NHG which doesn't map the packet's FC value, the packet will be dropped

## 2.2 Configuration and Management Requirements
- DSCP/EXP to FC maps must be allowed to be configured via the 2 CONFIG_DB tables with no requirement to be configurable via CLI.

## 2.3 Scalability Requirements
- Unchanged.

## 2.4 Warm Boot Requirements
- Unchanged - the new class based next hop group table must be compatible with existing warm boot requirements.

## 2.5 Restrictions
- fpmsyncd is not updated to use the new CLASS_BASED_NEXT_HOP_GROUP_TABLE as part of this enhancement. Anyone wishing to use this feature must use a modified version of fpmsyncd, or program the table directly.

# 3 Design
## 3.1 Overview
This design directly changes CONFIG_DB, APPL_DB, orchagent and sairedis.

## 3.2 DB Changes
### 3.2.1 APPL DB
Based on the next hop group split (https://github.com/Azure/SONiC/pull/712) on which this HLD is based on, a new CLASS_BASED_NEXT_HOP_GROUP table will be added to the APPL_DB with the following format:
```
### CLASS_BASED_NEXT_HOP_GROUP_TABLE
    ;Stores a list of FC-aware next hop groups.
    ;Status: Mandatory
    key           = CLASS_BASED_NEXT_HOP_GROUP_TABLE:string ; arbitrary string identifying the class based next hop group, as determined by the programming application.
    members       = NEXT_HOP_GROUP_TABLE.key,    ; one or more indexes within NEXT_HOP_GROUP_TABLE, separated by “,”
    class_map     = number:number,   ; one or more mapping from Forwarding Class to index in "members" field to use as NHG, separated by ","
```

Example:
    127.0.0.1:6379[1]> hgetall  "CLASS_BASED_NEXT_HOP_GROUP:CbfNhg1"
    1) "members"
    2) "Nhg1,Nhg2,Nhg3,Nhg4"
    3) "class_map"
    4) "0:0,1:0,2:1,3:1,4:2,5:2,6:3,7:3"

The ROUTE_TABLE is updated to allow the "nexthop_group" to allow both keys from NEXT_HOP_GROUP_TABLE and from the new CLASS_BASED_NEXT_HOP_GROUP_TABLE.
```
### ROUTE_TABLE
    ;Stores a list of routes
    ;Status: Mandatory
    key           = ROUTE_TABLE:prefix
    nexthop       = *prefix, ;IP addresses separated “,” (empty indicates no gateway)
    ifname        = *PORT_TABLE.key,   ; zero or more separated by “,” (zero indicates no interface)
    blackhole     = BIT ; Set to 1 if this route is a blackhole (or null0)
    nexthop_group = NEXT_HOP_GROUP_TABLE.key or CLASS_BASED_NEXT_HOP_GROUP_TABLE.key ; index within the NEXT_HOP_GROUP_TABLE or CLASS_BASED_NEXT_HOP_GROUP_TABLE, optionally used instead of nexthop and intf fields
 ```

 The LABEL_ROUTE_TABLE is updated to allow the "nexthop_group" to allow both keys from NEXT_HOP_GROUP_TABLE and from the new CLASS_BASED_NEXT_HOP_GROUP_TABLE.
```
### LABEL_ROUTE_TABLE
    ; Defines schema for MPLS label route table attributes
    ;Status: Mandatory
    key           = LABEL_ROUTE_TABLE:mpls_label ; MPLS label
    nexthop       = STRING                   ; Comma-separated list of nexthops.
    ifname        = STRING                   ; Comma-separated list of interfaces.
    weight        = STRING                   ; Comma-separated list of weights.
    nexthop_group = NEXT_HOP_GROUP_TABLE.key or CLASS_BASED_NEXT_HOP_GROUP_TABLE.key ; index within the NEXT_HOP_GROUP_TABLE or CLASS_BASED_NEXT_HOP_GROUP_TABLE, optionally used instead of nexthop and intf fields
 ```

### 3.2.2 CONFIG_DB
In order to store the DSCP/EXP to FC mappings, 2 new CONFIG_DB tables will be added:

```
### DSCP_TO_FC_MAP
    ;Stores a mapping between DSCP values and FC values. qos_map object with SAI_QOS_MAP_ATTR_TYPE == sai_qos_map_type_t::SAI_QOS_MAP_DSCP_TO_FC
    ;Status: Mandatory
    key        = DSCP_TO_FC_MAP_TABLE:string ; arbitrary string identifying the name of the map.
    dscp_value = 1*DIGIT
    fc_value   = 1*DIGIT
```

Example:
    127.0.0.1:6379> hgetall "DSCP_TO_FC_MAP_TABLE:AZURE"
     1) "3" ;dscp
     2) "3" ;fc
     3) "6"
     4) "5"
     5) "7"
     6) "5"
     7) "8"
     8) "7"
     9) "9"
    10) "8"

```
### EXP_TO_FC_MAP
    ;Stores a mapping between EXP values and FC values. qos_map object with SAI_QOS_MAP_ATTR_TYPE == sai_qos_map_type_t::SAI_QOS_MAP_EXP_TO_FC
    ;Status: Mandatory
    key        = EXP_TO_FC_MAP_TABLE:string ; arbitrary string identifying the name of the map.
    exp_value  = 1*DIGIT
    fc_value   = 1*DIGIT
```

Example:
    127.0.0.1:6379> hgetall "EXP_TO_FC_MAP_TABLE:AZURE"
     1) "3" ;exp
     2) "3" ;fc
     3) "6"
     4) "5"
     5) "7"
     6) "5"
     7) "8"
     8) "7"
     9) "9"
    10) "8"

## 3.3 Switch State Service Design
### 3.3.1 Orchestration Agent

A new orchestration agent will be written to handle the requests to both NEXT_HOP_GROUP_TABLE and CLASS_BASED_NEXT_HOP_GROUP_TABLE while also providing a common API for the route orchestration agent to use when working with next hop groups stored in these tables.

For a new entry in CLASS_BASED_NEXT_HOP_GROUP_TABLE, the orchestration agent will validate the data and create a new next hop group object in ASIC_DB of type SAI_NEXT_HOP_GROUP_TYPE_CLASS_BASED to which it will add the provided members as long as they have alreaedy been created in ASIC_DB. If an error occurs during this process and it's coming from the validation process, the task will be efectively removed from the process queue as an update to the entry would be needed in order to fix it; otherwise the task will be kept in the process queue for the event of the missing member(s) being created which would allow the class based next hop group to be created.

If the dataplane doesn't have any more room for a new next hop group object, the task will remain in the process queue for the event of space being freed.

There is a special scenario for creating the class based next hop groups, and that is when it references temporary next hop groups (as described in https://github.com/Azure/SONiC/pull/712), as these may be updated at some point which in turn will change their SAI ID. For this scenario, the class based next hop groups will keep a list of their temporary members and periodically check if it's SAI ID has been updated. If so, the SAI_NEXT_HOP_GROUP_MEMBER_ATTR_NEXT_HOP_ID attribute of the class based next hop group member will be updated to the match the new value and if the next hop group was updated to a proper (;-temporary) next hop group object, it will be erased from the specified list. When all the temporary next hop groups have been updated to proper next hop groups, the class based one will stop checking periodically for the updates.

For an updated entry in CLASS_BASED_NEXT_HOP_GROUP_TABLE, the orchestration agent will remove the group's previous members and add the updated ones. We do this due to the limitation of the SAI_NEXT_HOP_GROUP_MEMBER_ATTR_INDEX attribute which is CREATE_ONLY and so can't be updated. Instead of accounting for all the possibilities for the index of a member to be updated (by moving it to a different position in the list, removing a member that comes before it or adding a new one before it) which would be exhaustive to handle, we prefer this simpler and more robust solution. The class map will also be updated in ASIC_DB if necessary.

For a removed entry from CLASS_BASED_NEXT_HOP_GROUP_TABLE, the orchestration agent will remove the group from ASIC_DB only if it is not referenced anymore by other objects (such as routes).

Thanks to the common API provided by the new next hop group orhcestration agent, the route orchestration agent will not need any major updates in order for routes to work with both class based next hop groups and normal next hop groups. In order for this common API to work properly, the application(s) programming the NEXT_HOP_GROUP_TABLE and CLASS_BASED_NEXT_HOP_GROUP_TABLE must ensure there is no clash between the keys of the two tables. If such a clash exists, the non-CBF next hop group will be used and returned to the route orchestration agent.

The QoS orchestration agent is extended in order to process the DSCP_TO_FC_MAP_TABLE and EXP_TO_FC_MAP_TABLE entries. It's similar in functionality with the QoS task handling with the exception of the SAI_QOS_MAP_TYPE used for the entries created into ASIC_DB, being one of the SAI_QOS_MAP_TYPE_DSCP_TO_FORWARDING_CLASS or SAI_QOS_MAP_TYPE_MPLS_EXP_TO_FORWARDING_CLASS.

## 3.4 sairedis
Sairedis support has been added for objects of type "sai_map_t" for validation, serialization and deserialization in order for the "class_map" property of the class based next hop groups to work properly and also "fc" has been added to "sai_qos_map_params_t" object to support the DSCP/EXP to FC mappings.

## 3.5 SAI
The SAI changes are handled in https://github.com/opencomputeproject/SAI/pull/1193.

## 3.6 CLI
There is no requirement for adding CLI support for this feature.

# 4 Warm Boot Support
Unchanged.

# 4.1 Warm Upgrade
Unchanged.

# 5 Unit Test
Unit tests have been added to orchagent to test the mainline scenarios (create/delete) for the DSCP/EXP_TO_FC_MAP entries (update doesn't seem to be supported by CONFIG_DB tables) and mainline scenarios (create/update/delete) for the CLASS_BASED_NEXT_HOP_GROUP entries. Along with these mainline scenarios, different corner case scenarios have been tested as well, among which next hop group objects exhaustion has been tested to confirm the expected behaviour of class based next hop groups with limited resources available.

A unit test has also been added to sairedis to test the serialization and deserialization of the "sai_map_t" objects.

There are plans to complete the code coverage for the new orchagent code in the following month, as well as adding sonic-mgmt unit tests in the near future.
