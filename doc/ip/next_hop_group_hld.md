
# Routing and Next Hop Table Enhancement
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
          * [3.2.1 APP DB](#321-app-db)
      * [3.3 Switch State Service Design](#33-switch-state-service-design)
          * [3.3.1 Orchestration Agent](#331-orchestration-agent)
      * [3.4 Syncd](#34-syncd)
      * [3.5 SAI](#35-sai)
      * [3.6 CLI](#36-cli)
          * [3.6.1 Show Commands](#361-show-commands)
  * [4. Warm Boot Support](#4-warm-boot-support)
      * [4.1 Warm Upgrade](#31-warm-upgrade)
  * [5. Unit Test](#5-unit-test)


# Revision
| Rev |     Date    |       Author            | Change Description                         |
|:---:|:-----------:|:-----------------------:|--------------------------------------------|
| 0.1 | 25/11/2020  |   Thomas Cappleman      | Initial version                            |


# About this Manual
This document provides general information about an enhancement to the internal APP_DB routing table to split next hop information out into its own table.

Throughout this document a next hop group may refer to either a single next hop or a group of several next hops.

# 1 Introduction
Currently the route table within APP_DB contains all next hop information for a route embedded in that route's entry in the table.

At high scale (particularly when handling millions of routes all routed over multiple next hops) this is inefficient both in terms of performance and occupancy.  A given route entry needs to be built up including all its next hop information, which gets sent to and stored in APP_DB.  Orchagent then needs to receive, parse and handle all the information in that route entry every time.

A more efficient system would involve managing the next hop groups in use by the route table separately, and simply have the route table specify a reference to which next hop group to use.  Since at scale many routes will use the same next hop groups, this requires much smaller occupancy per route, and so more efficient building, transmission and parsing of per-route information.

# 2 Requirement Overview
## 2.1 Functional Requirements

- Improve efficiency of storage of routing information in APP_DB.
- Reduce time to reprogram routes when next hop information changes.

## 2.2 Configuration and Management Requirements
- The existing 'show ip route' and 'show ipv6 route' commands must be supported when using the new next hop group table.

## 2.3 Scalability Requirements
- Unchanged in requirements, but will improve scalability in terms of storage and programming time.

## 2.4 Warm Boot Requirements
- Unchanged - the new next hop group table must be compatible with existing warm boot requirements.

## 2.5 Restrictions
- fpmsyncd is not updated to use the new NEXT_HOP_GROUP_TABLE as part of this enhancement. Anyone wishing to use this feature must use a modified version of fpmsyncd, or program the table directly.

# 3 Design
## 3.1 Overview
This design directly changes APP_DB and orchagent, along with minor changes to support the existing CLI when using the new APP_DB table.

## 3.2 DB Changes
### 3.2.1 APP DB
Currently the ROUTE_TABLE in APP_DB includes all next hop information for the route, as follows:
```
### ROUTE_TABLE
    ;Stores a list of routes
    ;Status: Mandatory
    key           = ROUTE_TABLE:prefix
    nexthop       = *prefix, ;IP addresses separated “,” (empty indicates no gateway)
    ifname        = *PORT_TABLE.key,   ; zero or more separated by “,” (zero indicates no interface)
    blackhole     = BIT ; Set to 1 if this route is a blackhole (or null0)
 ```

This design adds a new NEXT_HOP_GROUP_TABLE, to store next hop group information to be used by one or more routes.
```
### NEXT_HOP_GROUP_TABLE
    ;Stores a list of groups of one or more next hops
    ;Status: Mandatory
    key           = NEXT_HOP_GROUP_TABLE:string ; arbitrary string identifying the next hop group, as determined by the programming application.
    nexthop       = *prefix,           ; IP addresses separated “,” (empty indicates no gateway)
    ifname        = *PORT_TABLE.key,   ; zero or more separated by “,” (zero indicates no interface)
```
Note that the identifier for a next hop group is entirely the decision of the programming application. Whether this is done randomly or algorithmically is up to the application - this design imposes no requirements on this.

The ROUTE_TABLE is then extended to allow a reference to the next hop group to be specified, instead of the current nexthop and intf fields.
```
### ROUTE_TABLE
    ;Stores a list of routes
    ;Status: Mandatory
    key           = ROUTE_TABLE:prefix
    nexthop       = *prefix, ;IP addresses separated “,” (empty indicates no gateway)
    ifname        = *PORT_TABLE.key,   ; zero or more separated by “,” (zero indicates no interface)
    blackhole     = BIT ; Set to 1 if this route is a blackhole (or null0)
    nexthop_group = NEXT_HOP_GROUP_TABLE:key ; index within the NEXT_HOP_GROUP_TABLE, optionally used instead of nexthop and intf fields
 ```

## 3.3 Switch State Service Design
### 3.3.1 Orchestration Agent

A new orchestration agent will be written to handle the new NEXT_HOP_GROUP_TABLE in APP_DB. For a new or updated entry in the NEXT_HOP_GROUP_TABLE, programming will depend on whether the group is configured with 1 or multiple next hops.

 - If the group has a single next hop, the next hop group orchagent will simply get the SAI identifier for that next hop from the neighbor orchagent, and use that as the SAI identifier for the group.
 - If the group has multiple next hops, the next hop group orchagent will add a next hop group to ASIC_DB and then add a next hop group member to ASIC_DB for every member of that group that is available to be used. Changes to the membership of the group will result in next hop group members being added to or removed from ASIC_DB. The next hop group orchagent will then maintain an association between the identifer of the group from APP_DB and the SAI identifier assigned to the next hop group.

Dummy code for this logic:
```
if (num_next_hops == 1)
	sai_id = sai ID of the next hop
else
	create next hop group
	add next hop group member for each next hop
	sai_id = sai ID of the next hop group
```

The handling of whether a next hop group member can be programmed or not, interfaces going up and down, etc. will match the same handling done by next hop groups managed by the route orchagent.

When the next hop group is deleted, the next hop group orchagent will remove the next hop group from the ASIC_DB if required, and remove the association between the group identifier from APP_DB and the SAI identifier from ASIC_DB.  This will only happen once the next hop group is no longer referenced by any routes - and so the next hop group will maintain a reference count within orchagent to ensure this. If orchagent restarts before all referencing routes are updated/deleted, the remaining routes will simply be unable to be programmed into ASIC_DB after restart. This will just be a transient state until the routes are updated in the ROUTE_TABLE.

The route orchagent will parse the new nexthop_group field, and call into the next hop group orchagent to find the corresponding SAI object ID to program as the next hop attribute.  If the next hop group does not exist, the route will remain on the list of routes to sync, and will be retried on future runs of the route orchagent.

If a next hop group cannot be programmed because the data plane limit has been reached, one next hop will be picked to be temporarily used for that group.  When a route is then programmed using that next hop group, it will be informed that a temporary form of the next hop group is in use, and so the route will remain on the pending list and will retry until the correct next hop group is programmed.  This mirrors the current behaviour in the route orchagent when the next hop group limit is reached. The APP_DB entries remain unchanged.

A few notes about this design:

- The next hop groups created by the next hop group orchagent will not interact with those created by the route orchagent's current mechanisms. This is because the route orchagent's groups are identified by the members of the group, and so changing those members makes it a different group, whereas the new next hop group orchagent's groups have an arbitrary index, with members able to change without it becoming a different group. Therefore even if a group in each category share the same members, they will be programmed separately into ASIC_DB, in case the members change. The expectation is that all routes will be programmed into APP_DB using the old next hop schema or the new one, and so the only occasion where next hop groups could overlap is if a next hop group is created for an ACL that overlaps with one created by the next hop group orchagent.
- A ROUTE_TABLE entry with both next hop information embedded and a reference to the NEXT_HOP_GROUP_TABLE will be ignored.
- Next hop groups created by the fine grained next hop group orchagent will be unchanged by this design.

## 3.4 SyncD
No change.


## 3.5 SAI
No changes are being made in SAI. The end result of what gets programmed over the SAI will be the same as currently gets programmed - the difference is the programming sequence to get there.

## 3.6 CLI

### 3.6.1 Show Commands

The output of 'show ip route' and 'show ipv6 route' will remain unchanged - the CLI code will resolve the next hop group ID referenced in the ROUTE_TABLE to display the next hops for the routes.

# 4 Warm Boot Support

Warm boot support must not be affected by this enhancement. Specifically, this means that (assuming the required routes don't change during warm boot), routes must be associated with the same next hop group after warm boot. The application programming APP_DB must therefore ensure that next hop groups get assigned the same identifier after reboot. This could be done by the application maintaining that information itself, or retrieving the identifiers from APP_DB at the start of warm boot.

Note that fpmsyncd is not being changed to use this enhancement, and so no open source application is affected by this, and a detailed design for how applications might choose to manage this is outside the scope of this design.

Fast reboot is not affected by this change. The requirements on the BGP application in terms of graceful restart remain the same - the only affect this change has is to give an alternative approach for what the BGP application programs into APP_DB.

# 4.1 Warm Upgrade

Existing applications will not be affected by this enhancement, so no warm upgrade support is needed. If in future an existing application is changed to use this feature, then a further enhancement should be made to handle warm upgrade. Adding warm upgrade support now would be untestable, due to no application using that code, and so is outside the scope of this design.

# 5 Unit Test

1. Verify that next hop group is added/updated/removed in hardware when next hop group table entry is added/updated/removed (for both a single next hop and a multiple next hop group).
2. Verify that a route with a reference to the next hop group table get programmed into hardware (for the group having a single next hop and multiple next hops).

The sonic-mgmt pytests will be extended to cover the new form of APP_DB.
