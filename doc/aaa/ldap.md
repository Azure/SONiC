# LDAP Name Service

## High Level Design Document
#### Rev 0.13

# Table of Contents
  * [List of Tables](#list-of-tables)
  * [Overview](#overview)
  * [Revision](#revision)
  * [Requirements](#requirements)
  * [Functional Description](#functional-description)
    * [LDAP NSS](#ldap-nss)
    * [LDAP Authentication](#ldap-authentication)
    * [LDAP Authorization](#ldap-authorization)
      * [LDAP Logon Authorization](#ldap-logon-authorization)
      * [LDAP Sudoers Authorization](#ldap-sudoers-authorization)
    * [Session Setup](#session-setup)
  * [Design](#design)
    * [Block Diagram of PAM and NSS Configuration](#block-diagram-of-pam-and-nss-configuration)
    * [Sequence Diagram of TACPLUS or RADIUS Authentication and LDAP Name Service](#sequence-diagram-of-tacplus-or-radius-authentication-and-ldap-name-service)
    * [ConfigDB Schema](#configdb-schema)
    * [OC YANG](#oc-yang)
    * [Klish](#klish)
    * [Click](#click)
    * [Known Issues and Notes](#known-issues-and-notes)
  * [Use Case](#use-case)
  * [Flow Diagrams](#flow-diagrams)
  * [Error Handling](#error-handling)
  * [Serviceability and Debug](#serviceability-and-debug)
  * [Warm Boot Support](#warm-boot-support)
  * [Scalability](#scalability)
  * [Unit Test](#unit-test)
    * [Prerequisites](#prerequisites)
    * [UT Authentication LDAP](#ut-authentication-ldap)
    * [UT Authentication TACPLUS with Name Service LDAP](#ut-authentication-tacplus-with-name-service-ldap)
    * [UT Authentication RADIUS with Name Service LDAP](#ut-authentication-radius-with-name-service-ldap)
    * [UT Authentication RADIUS with Logon Authorization LDAP and Name Service LDAP](#ut-authentication-radius-with-logon-authorization-ldap-and-name-service-ldap)
    * [UT Authentication RADIUS with Name Service LDAP and sudo Authorization LDAP](#ut-authentication-radius-with-name-service-ldap-and-sudo-authorization-ldap)
    * [UT Authentication Non Global NSS PAM SUDO options](#ut-authentication-non-global-nss-pam-sudo-options)
  * [References](#references)
    * [nss_ldap](#nss_ldap)
    * [pam_ldap](#pam_ldap)
    * [sudo_ldap](#sudo_ldap)
    * [Linux-PAM](#Linux-PAM)
    * [NSS Reference](#NSS-Reference)
    * [LDAP client configuration file](#LDAP-client-configuration-file)
    * [TACACS+ Authentication](#TACPLUS-Authentication)
    * [RADIUS Management User Authentication](#RADIUS-Management-User-Authentication)

# List of Tables

# Revision

| Rev | Date         |  Author            | Change Description                |
|:---:|:------------:|:------------------:|-----------------------------------|
| 0.1 | 03/20/2020   |  Arun Barboza      | Initial version                   |
| 0.2 | 03/23/2020   |  Arun Barboza      | After Internal Review             |
| 0.3 | 04/03/2020   |  Arun Barboza      | First External Review             |
| 0.4 | 04/29/2020   |  Arun Barboza      | Detail Config DB Table schema     |
| 0.5 | 04/30/2020   |  Arun Barboza      | Updated Unit Test Details         |
| 0.6 | 05/04/2020   |  Arun Barboza      | Fixed typos, omissions, examples  |
| 0.7 | 05/06/2020   |  Arun Barboza      | Updates for Future Release        |
| 0.8 | 05/07/2020   |  Arun Barboza      | Annotations in LDAP table         |
| 0.9 | 05/11/2020   |  Arun Barboza      | LDAP New Mgmt if- Balachandar M.  |
|     |              |                    | Update for VRF, SRC IP - Suresh R.|
| 0.10| 05/17/2020   |  Arun Barboza      | KLISH show aaa|ldap output draft. |
|     |              |                    | Added missing retry to ldap host. |
| 0.11| 05/17/2020   |  Arun Barboza      | Update the UT with KLISH examples.|
| 0.12| 05/20/2020   |  Arun Barboza      | IIFR, clarifications, OCYang, Maps|
| 0.13| 06/22/2020   |  Arun Barboza      | project-arlo review updates       |
|     |              |                    |                                   |

# Overview

Lightweight Directory Access Protocol (LDAP) allows a client to access
information stored in distributed directory services.

Name Service information typically includes users, hosts, groups. This
information is historically stored in flat files or other information
services (like a directory service).

This document describes the high level design of LDAP Name service feature
in SONiC.

# Requirements

- Obtain User Information from an LDAP server.

- Enable LDAP to be used as an authentication service.

- Enable LDAP to be used as an authorization service.

- Allow an authentication service, authorization service, separate from
  Name Service.

# Functional Description

## LDAP NSS

LDAP can be used as an option in the Name Services Switch(NSS) configuration.
The NSS configuration enables various programming APIs to use other sources
than the default files (e.g., Use LDAP directory information instead of
/etc/passwd for user and group information). User information includes uid,
gid, and home directory.

## LDAP Authentication

The LDAP Pluggable Authentication Module (PAM) can be used to authenticate
a CLI (SSH, or console) user to a Linux device like SONiC.

## LDAP Authorization

### LDAP Logon Authorization

The LDAP PAM can be used to authorize a user at login. (e.g., ensure the
user logging on is a member of a group aka group-based logon authorization).
This group can represent any membership (e.g., group of users who are
permitted to logon to a host). In PAM, authorization task is done by account
management modules.

### LDAP Sudoers Authorization

The sudoers policy plugin determines a user's sudo privileges. This policy
is driven by the /etc/sudoers file. LDAP can be used to drive this policy
by being a source in the NSS configuration. (e.g. can a user be allowed to
execute the click config command?)

## Session Setup

When a user logs into a SONiC device, there would need to be some session
environment (e.g. home directory). This can also be provided by PAM session
modules. The pam_open_session(3) mentions tasks such as creating or
mounting a home directory.

As of writing, the KLISH certificate to authenticate with the REST server
may not be necessary if the move to UNIX domain sockets from TCP sockets
takes place.

In order to perform RBAC (Role Based Access Control), role(s) may be
needed for the user. The PAM session module could retrieve the same and
store it in the redis ConfigDB UserTable. Some options to retrieve roles.

- Infer from the user's group membership. A possible way would be to use membership in a group to indicate the role membership (say by using a map).
- Have a role attribute in the user's LDAP entry. This might require an LDAP schema change so could be less desirable.

The session modules could be made extensible by passing configuration options.


# Design

Publicly available pre-built debian packages for libpam-ldap, libnss-ldap,
and sudo-ldap, will be planned on being used for providing the backend.
Configuration will take place through the Host Config Daemon using
configuration stored in ConfigDB. The ConfigDB tables can be set using
REST/gNMI, KLISH or click.


## Block Diagram of PAM and NSS Configuration


```
           +-------+  +---------+
           |  SSH  |  | Console |
           +---+---+  +----+----+
               |           |
               |   (4)     |
               |           |
               |           |
               V           V
+---------------------------------------------+
|    AUTHENTICATION,            Name          |
|     AUTHORIZATION,           Service        |
|    & SESSION SETUP                          |
| +-------------------+ +-------------------+ |
| | PAM Configuration | | NSS Configuration | |
| |       Files       | |       Files       | |
| +-------------------+ +-------------------+ |
|           ^                     ^           |        +---------------------+
|           |                     |           |        |                     |
|           |                     |           |  (3)   |  +------------+     |
|           +---------------------+-----------+--------+--+ AAA Config |     |
|                                             |        |  +------------+     |
|                                             |        |                     |
|    +--------------+       +------------+    |        | Host Config Daemon  |
|    |     PAM      |       |    NSS     |    |        +----------^----------+
|    |  Libraries   |       |  Libraries |    |                   |
|    +------------- +       +------------+    |                   |
+---------------------------------------------+               (2) |
                                                                  |
                                                                  |
           +---------+                                    +-------+--------+
           |         |                (1)                 |                |
           |   CLI   +------------------------------------>    ConfigDB    |
           |         |                                    |                |
           +---------+                                    +----------------+

```

1. The CLI is used to update the ConfigDB with the LDAP and AAA configuration.
2. The Host Config Daemon(HCD) reads data from ConfigDB.
3. The HCD configures the PAM & NSS configuration files. (The AAA config
   module in the HCD is responsible for modifying the configuration files.)
4. All new CLI sessions (SSH, Console), now use the new PAM & NSS
   configurations to obtain name service information, authenticate, and
   authorize users.

## Sequence Diagram of TACPLUS or RADIUS Authentication and LDAP Name Service

Following is the sequence of events during TACACS+/RADIUS authenticated login
for an SSH client, when LDAP is used for name service:

```
SSH Client              SONiC Device                     TACACS+ Server
                  SSH Server     NSS LDAP Plugin         or RADIUS Server
----------------------------------------------------------------------

    |                 |                 |                    |
    |                 |                 |                    |
    |  Login(1)       |                 |                    |
    +---------------->|                 |                    |
    |                 | getpwnam_r()(2) |                    |
    |                 +---------------->|    Retrieve        |
    |                 |                 |(3) data from LDAP  |
    |                 | user info.(4)   |    or name service |
    |                 |<----------------+    cache daemon    |
    |                 |                 |                    |
    |        (5)      |                 |                    |
    |  pam_auth_rqst. |                 |                    |
    |  to PAM library |                 |                    |
    |                 |                                      |
    |                 |   TACACS+ Authentication START (6)   |
    |                 |   or RADIUS Access-Request           |
    |                 +------------------------------------->|
    |                 |                                      |
    |                 |   Authentication REPLY - PASS (7)    |
    |                 |   or RADIUS Access-Accept            |
    |                 |<-------------------------------------+
    |                 |                                      |
    |                 |                 |                    |
    |  Success(8)     |                 |                    |
    |<----------------+                 |                    |
    |                 |                 |                    |
```

1. A user tries to login through a SSH client.
2. A getpwnam_r() call is made to obtain user information.
3. If the user is not found in the local /etc/passwd file, the NSS
   LDAP plugin retreives the data from the LDAP server (or name service
   cache daemon). If not found, login fails.
4. User information is returned to the SSH server.
5. The SSH server makes a PAM Authentication request to the PAM library.
6. The PAM configuration causes a Authentication-Start to TACACS+ server.
   (or RADIUS Access-Request to RADIUS server).
7. TACACS+ returns Authentication-Reply (PASS) (or RADIUS returns Access-
   Accept) -- (Goto 8.),
   If Authentication-Reply (FAIL)/Access-Reject, the user authentication has
   failed. End.
8. The user has authenticated successfully.

Instead of TACACS+, RADIUS (or LDAP) could be used as the authentication
service as well.

## ConfigDB Schema


There will be New Config DB Table schema for holding LDAP Configuration information.

- LDAP                 (keys: 'global' [|'nss' | 'sudo' | 'pam'])

- LDAP_SERVER          (keys: <server_1> .... <server_n>)

- LDAP_MAP             (keys[0]: ATTRIBUTE,
                                 OBJECTCLASS,
                                 DEFAULT_ATTRIBUTE_VALUE,
                                 OVERRIDE_ATTRIBUTE_VALUE
                        keys[1]: <attribute|objectclass>))

An existing table will be modified to allow for LDAP Authentication,
Authorization, Name Service, and Session Setup.

- AAA                  (new keys: 'authorization', 'nss' [, 'open_session']?)

  ? The 'open_session' key will be reserved for Implementation In a Future
    Release(IIFR)


```
        #--------------------------------------------------#
        #                                                  #
        #       LDAP                     LDAP_SERVER       #
        #                                                  #
        #--------------------------------------------------#
        #                                                  #
        #              ____ nss      ....  server_1        #
        #              |             ....                  #
        #  global -----+--- sudo     ....                  #
        #              |___          ....                  #
        #                   pam      ....  server_n        #
        #                                                  #
        #--------------------------------------------------#

        #--------------------------------------------------#
        #                                                  #
        #       LDAP_MAP_xyz                               #
        #                                                  #
        #--------------------------------------------------#

        #--------------------------------------------------#
        #                                                  #
        #      AAA                                         #
        #                                                  #
        #--------------------------------------------------#
        #                                                  #
        #  authentication                                  #
        #                                                  #
        #  authorization                                   #
        #                                                  #
        #  nss                                             #
        #                                                  #
        #  [open_session]?                                 #
        #                                                  #
        #  ? For Implementation In a Future Release (IIFR) #
        #                                                  #
        #--------------------------------------------------#

```

### LDAP Table Schema

This table holds some global LDAP configuration values common to NSS, PAM,
and sudoers. (e.g. search base in LDAP)

```
; LDAP configuration attributes global to the device. Upto 4 rows can exist in
; the table. Any of the "global" row LDAP attributes can be overriden by the
; server use_type row settings in the same table.
; Only a few settings can be overriden in the per server Table (please see the
; next table)
; Key
global_key           = "global" / "nss" / "sudo" / "pam";
                                 ;
                                 ;   Attributes supported in all use_types
                                 ;   unless noted Eg:
                                 ;      n --> "nss" only
                                 ;      s --> "sudo" only
                                 ;      p --> "pam" only
                                 ;      - --> "global" support only.
                                 ;      np --> "nss" and "pam" only
                                 ;
                                 ;   (Units,Default[ - meaning])[([snp]*)]
                                 ;   Eg: (seconds,(0 - indefinite))
                                 ;       means that
                                 ;         the units are in seconds,
                                 ;         the default when the attribute is
                                 ;           absent is 0, which means wait
                                 ;           indefinitely.
                                 ;         the attribute is supported on all
                                 ;         use_types.

; Attributes
timelimit            = 1*nDIGIT  ; Search time limit (secs,(0 - indefinite))
bind_timelimit       = 1*nDIGIT  ; Connect time limit (secs,(10))
idle_timelimit       = 1*nDIGIT  ; NSS idle time limit (secs,(0 - indefinite))(n)
retry                = 1*2DIGIT  ; Retry (attempts,(0,none))
port                 = 1*5DIGIT  ; Port (number,(389))
scope                = 1*nVCHAR  ; Search scope ("sub"|"one"|"base",("sub"))(np)
ldap_version         = 1DIGIT    ; LDAP protocol version (2|3,(3))
base                 = 1*nVCHAR  ; Search base DN
ssl                  = 1*nVCHAR  ; Use TLS? ("on"|"off"|"start_tls",("off"))
binddn               = 1*nVCHAR  ; Bind DN (,( - bind anonymously))
bindpw               = 1*nVCHAR  ; Bind credentials (,( - bind anonymously))
pam_filter           = 1*nVCHAR  ; Filter for retrieving user info. (,)(p)
pam_login_attribute  = 1*nVCHAR  ; Attribute for retrieving user info.
                                 ; (,("uid"))(p)
pam_group_dn         = 1*nVCHAR  ; DN of a group for login Authorization (,)(p)
pam_member_attribute = 1*nVCHAR  ; Attribute for login Authorization (,)(p)
sudoers_base         = 1*nVCHAR  ; Sudo LDAP queries search base DN (,)(s)
nss_base_passwd      = 1*nVCHAR  ; <basedn?scope?filter> ... (,)(np)
nss_base_group       = 1*nVCHAR  ; <basedn?scope?filter> ... (,)(n)
nss_base_shadow      = 1*nVCHAR  ; <basedn?scope?filter> ... (,)(n)
nss_base_netgroup    = 1*nVCHAR  ; <basedn?scope?filter> ... (,)(n)
nss_base_sudoers     = 1*nVCHAR  ; <basedn?scope?filter> ... (,)(n)
nss_initgroups_ignoreusers = 1*nVCHAR  ; NOT_FOUND for initgroups(3) (,)(n)
src_ip               = IPAddress ; source IPv4|IPv6 addr for LDAP client (,)(-)
vrf                  = 1*nVCHAR  ; vrf_name (,)(-)

```

### LDAP_SERVER Table Schema

This table holds per server configuration (e.g. connection retries)
```
; LDAP per server configuration on the device.
; Key
server_key           = NameOrIPAddress;  LDAP server's DNS name or IPv4|6 addr
; Attributes
port                 = 1*5DIGIT  ; per server Port
use_type             = 1*nVCHAR  ; "all" / "nss" / "sudo" / "pam" ("all")
priority             = 1*2DIGIT  ; specify LDAP server's priority (1)
ssl                  = 1*nVCHAR  ; Use TLS? ("on"|"off"|"start_tls",("off"))
retry                = 1*2DIGIT  ; Retry (attempts,(0,none))

```

### LDAP_MAP Table Schema

This table holds the maps for LDAP configuration.

- LDAP_MAP_ATTRIBUTE
- LDAP_MAP_OBJECTCLASS
- LDAP_MAP_DEFAULT_ATTRIBUTE_VALUE
- LDAP_MAP_OVERRIDE_ATTRIBUTE_VALUE

```
; LDAP_MAP
; Key
map_name             = "ATTRIBUTE"/
                       "OBJECTCLASS"/
                       "DEFAULT_ATTRIBUTE_VALUE"/
                       "LDAP_MAP_OVERRIDE_ATTRIBUTE_VALUE" ; Map Name
from_key             = 1*nVCHAR  ;  Map key
; Attributes
to                   = 1*nVCHAR  ;  Map value
```

### AAA Table Schema

This is an existing table which will hold additional keys for configuring
NSS, and Authorization. For Authentication a key exists already.

```
; Key
aaa_key              = "authentication" / "authorization" / "nss" ; AAA type
; Attributes
; For "authentication" / "authorization" Key only
login                = LIST(1*32VCHAR)   ; "local"/"ldap"/"radius"/"tacacs+" ; PAM modules
failthrough          = "True" / "False"  ; failthrough mechanism for pam modules
debug                = "True" / "False"  ; Debugging (Developer)
trace                = "True" / "False"  ; Packet Trace (Developer)
; For "nss" Key only
passwd               = "login" / "ldap" / "radius" / "tacacs+"
group                = "login" / "local" / "ldap"
shadow               = "login" / "local" / "ldap"
netgroup             = "local" / "ldap"
sudoers              = "ldap"
; For "nss" Key: "login" ==> Name Service db is based on authentication login
```

## OC YANG

OC-Yang extensions need to be written for LDAP for the following:

- LDAP global configurations.
- LDAP per server configuration.
- LDAP Maps.
- AAA login authorization.
- AAA nss configuration.

The OC-Yang tree would need to be extended, and modified in the following
areas as follows: [ All new leaves ]
```
module: openconfig-system
  +--rw system
...
     +--rw aaa
...
     |  +--rw authentication
...
     |  |  +--rw config
     |  |  |  +--rw authentication-method* (This is existing leaf-list that
                                            needs to be extended for "ldap"
                                            option)
...
     |  +--rw oc-sys-ext:authorization
...
     |  |  +--rw oc-aaa-ldap-ext:login (New container: LDAP logon authorization)
     |  |     +--rw oc-aaa-ldap-ext:config
     |  |     |  +--rw oc-aaa-ldap-ext:authorization-method*   union
...
     |  +--rw oc-aaa-ldap-ext:name-service   (New container)
     |     +--rw oc-aaa-ldap-ext:passwd
     |        +--rw config
     |           +--rw oc-aaa-ldap-ext:name-service-method*
...
     |     +--rw oc-aaa-ldap-ext:shadow
     |        +--rw config
     |           +--rw oc-aaa-ldap-ext:name-service-method*
...
     |     +--rw oc-aaa-ldap-ext:group
     |        +--rw config
     |           +--rw oc-aaa-ldap-ext:name-service-method*
...
     |     +--rw oc-aaa-ldap-ext:netgroup
     |        +--rw config
     |           +--rw oc-aaa-ldap-ext:name-service-method*
...
     |     +--rw oc-aaa-ldap-ext:sudoers
     |        +--rw config
     |           +--rw oc-aaa-ldap-ext:name-service-method*
...
     |  +--rw server-groups
     |     +--rw server-group* [name]  (new LDAP_ALL group)
                                       (new LDAP_SUDO|PAM|NSS groups)
...
     |        +--rw servers
     |           +--rw server* [address]
     |        |     |
     |        |     |
...
     |        |     |
     |        |     +--rw oc-aaa-ldap-ext:ldap   (New container)
     |        |        +--rw oc-aaa-ldap-ext:config
     |        |        |  +--rw oc-aaa-ldap-ext:port?                  oc-inet:port-number
     |        |        |  +--rw oc-aaa-ldap-ext:use-type?              enumeration
     |        |        |  +--rw oc-aaa-ldap-ext:priority?              uint8
     |        |        |  +--rw oc-aaa-ldap-ext:ssl?                   ldap-ssl-type
     |        |        |  +--rw oc-aaa-ldap-ext:retransmit-attempts?   uint8
     |        |        +--ro oc-aaa-ldap-ext:state
     |        |           +--ro oc-aaa-ldap-ext:port?                  oc-inet:port-number
     |        |           +--ro oc-aaa-ldap-ext:use-type?              enumeration
     |        |           +--ro oc-aaa-ldap-ext:priority?              uint8
     |        |           +--ro oc-aaa-ldap-ext:ssl?                   ldap-ssl-type
     |        |           +--ro oc-aaa-ldap-ext:retransmit-attempts?   uint8
     |        +--rw oc-aaa-ldap-ext:ldap   (New Container)
     |           +--rw oc-aaa-ldap-ext:config
     |           |  +--rw oc-aaa-ldap-ext:source-interface?             -> /oc-if:interfaces/interface/name
     |           |  +--rw oc-aaa-ldap-ext:vrf-name?                     -> /oc-ni:network-instances/network-instance/name
     |           |  +--rw oc-aaa-ldap-ext:search-time-limit?            uint32
     |           |  +--rw oc-aaa-ldap-ext:bind-time-limit?              uint32
     |           |  +--rw oc-aaa-ldap-ext:retransmit-attempts?          uint8
     |           |  +--rw oc-aaa-ldap-ext:port?                         oc-inet:port-number
     |           |  +--rw oc-aaa-ldap-ext:version?                      uint8
     |           |  +--rw oc-aaa-ldap-ext:base?                         string
     |           |  +--rw oc-aaa-ldap-ext:ssl?                          ldap-ssl-type
     |           |  +--rw oc-aaa-ldap-ext:bind-dn?                      string
     |           |  +--rw oc-aaa-ldap-ext:bind-pw?                      string
     |           |  +--rw oc-aaa-ldap-ext:source-address?               oc-inet:ip-address
     |           |  +--rw oc-aaa-ldap-ext:vrf-name?                     string
     |           |  +--rw oc-aaa-ldap-ext:idle-time-limit?              uint32
     |           |  +--rw oc-aaa-ldap-ext:nss-base-group?               string
     |           |  +--rw oc-aaa-ldap-ext:nss-base-shadow?              string
     |           |  +--rw oc-aaa-ldap-ext:nss-base-netgroup?            string
     |           |  +--rw oc-aaa-ldap-ext:nss-base-sudoers?             string
     |           |  +--rw oc-aaa-ldap-ext:nss-initgroups-ignoreusers?   string
     |           |  +--rw oc-aaa-ldap-ext:pam-filter?                   string
     |           |  +--rw oc-aaa-ldap-ext:pam-login-attribute?          string
     |           |  +--rw oc-aaa-ldap-ext:pam-group-dn?                 string
     |           |  +--rw oc-aaa-ldap-ext:pam-member-attribute?         string
     |           |  +--rw oc-aaa-ldap-ext:sudoers-base?                 string
     |           |  +--rw oc-aaa-ldap-ext:scope?                        enumeration
     |           |  +--rw oc-aaa-ldap-ext:nss-base-passwd?              string
...
     |  |        +--rw oc-aaa-ldap-ext:maps
     |  |           +--rw oc-aaa-ldap-ext:map* [name from]
     |  |              +--rw oc-aaa-ldap-ext:name      -> ../config/name
     |  |              +--rw oc-aaa-ldap-ext:from      -> ../config/from
     |  |              +--rw oc-aaa-ldap-ext:config
     |  |              |  +--rw oc-aaa-ldap-ext:name?   enumeration
     |  |              |  +--rw oc-aaa-ldap-ext:from?   string
     |  |              |  +--rw oc-aaa-ldap-ext:to?     string
     |  |              +--ro oc-aaa-ldap-ext:state
     |  |                 +--ro oc-aaa-ldap-ext:name?   enumeration
     |  |                 +--ro oc-aaa-ldap-ext:from?   string
     |  |                 +--ro oc-aaa-ldap-ext:to?     string

...


```

## REST/gNMI

REST, and gNMI support would be needed by writing transformer annotation,
overloads, and SONiC Yang.

## KLISH

KLISH commands (XML, actioners, renderers) needs to be written for

- AAA authentication ldap option.

```
sonic(config)# [no] aaa authentication login default local \
                              [ | group { radius | tacacs+ | ldap } ]
sonic(config)# [no] aaa authentication login default \
                              group { radius | tacacs+ | ldap } [ local ]
sonic(config)# [no] aaa authentication login default
```

- AAA authorization login ldap option..

```
sonic(config)# [no] aaa authorization login default { group ldap | local }
```

- AAA name-service.

```
sonic(config)# [no] aaa name-service passwd { group ldap | login | local }
sonic(config)# [no] aaa name-service shadow { group ldap | login | local }
sonic(config)# [no] aaa name-service group  { group ldap | login | local }
sonic(config)# [no] aaa name-service netgroup { group ldap | local }
sonic(config)# [no] aaa name-service sudoers  { group ldap | local }
```

- LDAP global|nss|pam|sudo server authentication, authorization, name-service parameters.


```
ldap-server timelimit  <0 - 65535>
- The default value is 0 seconds.

ldap-server bind-timelimit <0 - 65535>
- The default value is 10 seconds.

ldap-server idle-timelimit  <0 - 65535>
- The default value is 0 seconds.

ldap-server retry  <0 - 10>
- Default value is 0.

ldap-server port <0 - 65535>
- Default value is 389.

ldap-server scope  [sub|one|base]
- Default value is "sub".

ldap-server ldap-version  [2|3]
- Default value is 3.

ldap-server base <string>

ldap-server ssl  [on|off|start_tls]
-Default value is "off"

ldap-server binddn  <string>

ldap-server bindpw <string>

ldap-server pam-filter <string>

ldap-server pam-login-attribute <string>
 -Default value is "uid".
 
ldap-server pam-group-dn <string>

ldap-server pam-member-attribute <string>

ldap-server sudoers-base  <string>

ldap-server nss-base-passwd  <string>

ldap-server nss-base-group <string>

ldap-server nss-base-shadow  <string>

ldap-server nss-base-netgroup <string>

ldap-server nss-base-sudoers <string>

ldap-server nss-initgroups-ignoreusers <string>

ldap-server source-ip <ip>

ldap-server vrf  <vrf-name>

====================================================================

LDAP server NSS commands:

ldap-server nss timelimit  <0 - 65535>
- The default value is 0 seconds.

ldap-server nss bind-timelimit <0 - 65535>
- The default value is 10 seconds.

ldap-server nss idle-timelimit  <0 - 65535>
- The default value is 0 seconds.

ldap-server nss retry  <0 - 10>
- Default value is 0.

ldap-server nss port <0 - 65535>
- Default value is 389.

ldap-server nss scope  [sub|one|base]
- Default value is "sub".

ldap-server nss ldap-version  [2|3]
- Default value is 3.

ldap-server nss base <string>

ldap-server nss ssl  [on|off|start_tls]
-Default value is "off"

ldap-server nss binddn  <string>

ldap-server nss bindpw <string>

ldap-server nss nss-base-passwd  <string>

ldap-server nss nss-base-group <string>

ldap-server nss nss-base-shadow  <string>

ldap-server nss nss-base-netgroup <string>

ldap-server nss nss-base-sudoers <string>

ldap-server nss nss-initgroups-ignoreusers <string>

=======================================================================

LDAP server PAM commands:

ldap-server pam timelimit  <0 - 65535>
- The default value is 0 seconds.

ldap-server pam bind-timelimit <0 - 65535>
- The default value is 10 seconds.

ldap-server pam retry  <0 - 10>
- Default value is 0.

ldap-server pam port <0 - 65535>
- Default value is 389.

ldap-server pam scope  [sub|one|base]
- Default value is "sub".

ldap-server pam ldap-version  [2|3]
- Default value is 3.

ldap-server pam base <string>

ldap-server pam ssl  [on|off|start_tls]
-Default value is "off"

ldap-server pam binddn  <string>

ldap-server pam bindpw <string>

ldap-server pam pam-filter <string>

ldap-server pam pam-login-attribute <string>
 -Default value is "uid".
 
ldap-server pam pam-group-dn <string>

ldap-server pam pam-member-attribute <string>

ldap-server pam nss-base-passwd  <string>


======================================================================

LDAP server sudo commands:

ldap-server sudo timelimit  <0 - 65535>
- The default value is 0 seconds.

ldap-server sudo bind-timelimit <0 - 65535>
- The default value is 10 seconds.

ldap-server sudo retry  <0 - 10>
- Default value is 0.

ldap-server sudo port <0 - 65535>
- Default value is 389.

ldap-server sudo ldap-version  [2|3]
- Default value is 3.

ldap-server sudo base <string>

ldap-server sudo ssl  [on|off|start_tls]
-Default value is "off"

ldap-server sudo binddn  <string>

ldap-server sudo bindpw <string>

ldap-server sudo sudoers-base  <string>


===============================================================================

LDAP server configuration commands:

ldap-server host <name|IPV4|IPV6 address>  use-type [all|nss|sudo|pam] | port <1-665535> | priority <1 - 99> ssl [on|off|start_tls]

- use-type [all|nss|sudo|pam] - Default value is "all"
- priority <1 - 99> -  Default value is 1
- ssl [on|off|start_tls] -  Default value is "off"
- retry  <1 - 10> - Default value is 0

===============================================================================

LDAP server MAP commands:

ldap-server map attribute <key> to <value>

Eg: ldap-server map attribute uid to "sAMAccountName"
      ldap-server map attribute shadowLastChange to "pwdLastSet"

ldap-server map objectclass <key> to <value>
ldap-server map default-attribute-value  <key> to <value>
ldap-server map override-attribute-value  <key> to <value>
```


- Show commands.

```
sonic# show ldap-server
---------------------------------------------------------
LDAP  Global Configuration
---------------------------------------------------------
binddn        : dc=sji,dc=broadcom,dc=net
--------------------------------------------------------------------------------
HOST            USE-TYPE PRIORITY SSL       RETRY
--------------------------------------------------------------------------------
1.1.1.1         -         1       -         -
sonic#

sonic# show aaa
---------------------------------------------------------
AAA Authentication Information
---------------------------------------------------------
failthrough  : True
login-method : ldap, local
sonic#

```

## Click

```
$ show aaa
$ show ldap
```


Similar to KLISH, but config, and show utilities.

## Known Issues and Notes

### Known Issues

- The publicly available pre-built debian packages for libpam-ldap, libnss-ldap,
and sudo-ldap, do not have source-ip or Vrf support. The same applies for the openldap-client library.

- authorization-sudoers exposes Linux/UNIX functionality, thus making it
  apparent that SONiC is Linux/UNIX based.

- The existing aaa authentication login-method KLISH cli, needs to be
  future proofed for allowing for separate console and ssh login methods.

### Implementation Notes

#### Authentication Login: local,ldap ; Failthrough: False

The LDAP user information is present in the *local* database (because with LDAP
authentication, *ldap* will be used as a *nss* source). Thus, *local*
authentication will be attempted for LDAP users, which will likely fail.
The recommendation is to enable *failthrough*.

#### Authentication Login: ldap,local ; Failthrough: False

For a local user that is not present in the LDAP, the authentication against
the *ldap* pam module will return a PAM_USER_UNKNOWN error, which is not
considered an authentication failure, thus authentication will take place
with the *local* (unix) pam module.

#### Per LDAP Server Configuration ; Failthrough: True

The OSS *pam_ldap* module can have only a single instance. Configuration
cannot be supported on a per individual server basis. An authentication failure
at a single LDAP server results in failure being returned for the entire
PAM stack. Please see [Linux-PAM](#Linux-PAM), under *config=<path>* --
"Configuring multiple  instances  of pam_ldap  for the same service with
different configuration files is not supported, because the configuration
information is cached."

### Implementation In a Future Release

Some part of the design mentioned in the document is for Implementation In a
Future Release and have been marked as such with ? (IIFR) annotations.

# Use Case

## LDAP Authentication and LDAP Name Service

LDAP provides the name services user, group information.
User's session is setup to create home directory locally, if it does not exist.
Users are authenticated through LDAP servers. 


```

   +------------+    +-----------+       +-----------------------+
   |            |    |           |       |                       |
   | SSH Client |<-->| SONiC DUT |<----->| LDAP Name Service &   |
   |            |    |           |       | Authentication Server |
   |            |    |           |       |                       |
   +------------+    +-----------+       +-----------------------+

```

```

sonic(config)# ldap-server base "dc=example,dc=com"
sonic(config)# ldap-server ssl start_tls
sonic(config)# ldap-server 10.59.1.9

; Use defaults for locating the bases of passwd, shadow, and group entries.

sonic(config)# aaa authentication failthrough enable
sonic(config)# aaa authentication login default group ldap local

; Use defaults for name-service based on login authentication 

```


## TACACS+ Authentication, LDAP Authorization, and LDAP Name Service

LDAP provides the name service user, group information.
User's session is setup to create home directory locally, if it does not exist.
Users are authenticated through TACACS+ servers. 
Users are authorized through LDAP servers for logon authorization only.
User's click interface is sudoers authorized through LDAP servers. (i.e
  the rules to check whether the user is allowed to execute the sudo
  command (config, show, sonic-clear) are obtained from LDAP servers.
  This does not apply to KLISH interface, since the authorization for
  that comes from the role assigned to the user).

```

                                         +----------------------------+
                                         |                            |
                                         | LDAP Name Service,   and   |
                                     +-->| LDAP sudoers Authorization |
                                     |   |          Servers           |
                                     |   |                            |
   +------------+    +-----------+   |   +----------------------------+
   |            |    |           |   |
   |            |    |           |   |   +----------------------------+
   |            |    |           |   |   |                            |
   | SSH Client |<-->| SONiC DUT |<--+-->| TACACS+   Authentication   |
   |            |    |           |   |   |         Servers            |
   |            |    |           |   |   +----------------------------+
   |            |    |           |   |
   |            |    |           |   |
   +------------+    +-----------+   |   +----------------------------+
                                     |   |                            |
                                     +-->| LDAP Login Authorization   |
                                         |         Servers            |
                                         +----------------------------+

```

```


; LDAP Name Service parameters
sonic(config)# ldap-server base "dc=example,dc=com"
sonic(config)# aaa name-service passwd group ldap
sonic(config)# aaa name-service shadow group ldap
sonic(config)# aaa name-service group group ldap

; Use this server for name-service and sudo-ldap authorization only, with one
; reconnect attempt
sonic(config)# ldap-server rsa.loc.example.net use-type nss,sudo ldap-retry 1


; Logon Authorization from LDAP
; Use this server for login authorization only
sonic(config)# ldap-server 10.59.1.9 use-type pam

sonic(config)# ldap-server pam-groupdn "cn=t1.loc.example.com,ou=hostaccess,dc=example,dc=com"
sonic(config)# ldap-server pam-member-attribute "uniqueMember"
sonic(config)# aaa authorization login default group ldap

; sudo-ldap authorization
sonic(config)# ldap-server sudoers-base "dc=example,dc=com"
sonic(config)# aaa name-service sudoers group ldap

; Authentication from TACACS+
sonic(config)# tacacs-server 10.59.1.15
sonic(config)# aaa authentication failthrough enable
sonic(config)# aaa authentication login default group tacacs+ local

```

# Flow Diagrams

Please see the diagrams in section [Design](#design)

# Error Handling

PAM and NSS modules return errors as per [Linux-PAM](#Linux-PAM), and
[NSS](#NSS-Reference) respectively.

# Serviceability and Debug

The LDAP PAM and NSS modules can be debugged using the "debug" option
of AAA. (i.e. config aaa debug enable)

# Warm Boot Support

N/A

# Scalability

A maximum of 8 LDAP servers can be configured.

# Unit Test

## Prerequisites

```
  LDAP Server with User, Group information.
  TACACS+/RADIUS Server with User passwords configure.
```

### LDAP data

For sudo authorization (*sudoRole* objectclass), the LDAP server schema
needs to be updated from DUT file /usr/share/doc/sudo-ldap/schema.OpenLDAP.
Please see next section for some tips.


```

$ ldapsearch -x -b "dc=sji,dc=broadcom,dc=net" 
# extended LDIF
#
# LDAPv3
# base <dc=sji,dc=broadcom,dc=net> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# sji.broadcom.net
dn: dc=sji,dc=broadcom,dc=net
objectClass: top
objectClass: dcObject
objectClass: organization
o: sji.broadcom.net
dc: sji

# admin, sji.broadcom.net
dn: cn=admin,dc=sji,dc=broadcom,dc=net
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
description: LDAP administrator

# Group, sji.broadcom.net
dn: ou=Group,dc=sji,dc=broadcom,dc=net
objectClass: organizationalUnit
ou: Group

# People, sji.broadcom.net
dn: ou=People,dc=sji,dc=broadcom,dc=net
objectClass: organizationalUnit
ou: People

# Sudoers, sji.broadcom.net
dn: ou=Sudoers,dc=sji,dc=broadcom,dc=net
objectClass: organizationalUnit
ou: Sudoers

# sudo, Group, sji.broadcom.net
dn: cn=sudo,ou=Group,dc=sji,dc=broadcom,dc=net
objectClass: top
objectClass: posixGroup
gidNumber: 27
memberUid: arun
memberUid: ldapuser
cn: sudo

# docker, Group, sji.broadcom.net
dn: cn=docker,ou=Group,dc=sji,dc=broadcom,dc=net
objectClass: top
objectClass: posixGroup
gidNumber: 999
memberUid: arun
memberUid: ldapuser
memberUid: raduser1
cn: docker

# arun, People, sji.broadcom.net
dn: cn=arun,ou=People,dc=sji,dc=broadcom,dc=net
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: arun
uid: arun
uidNumber: 1200
gidNumber: 1000
homeDirectory: /home/arun
loginShell: /bin/bash

# ldapuser, People, sji.broadcom.net
dn: cn=ldapuser,ou=People,dc=sji,dc=broadcom,dc=net
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: ldapuser
uid: ldapuser
uidNumber: 1201
gidNumber: 1000
homeDirectory: /home/ldapuser
loginShell: /bin/bash

# raduser1, People, sji.broadcom.net
dn: cn=raduser1,ou=People,dc=sji,dc=broadcom,dc=net
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: raduser1
uid: raduser1
uidNumber: 1202
gidNumber: 100
homeDirectory: /home/raduser1
loginShell: /bin/bash

# role1, Sudoers, sji.broadcom.net
dn: cn=role1,ou=Sudoers,dc=sji,dc=broadcom,dc=net
objectClass: sudoRole
objectClass: top
cn: role1
sudoUser: raduser1
sudoHost: ALL
sudoCommand: ALL
sudoRunAsUser: ALL
sudoRunAsGroup: ALL

# search result
search: 2
result: 0 Success

# numResponses: 12
# numEntries: 11
$ 

```

### LDAP sudo-ldap.schema addition

For 2.4.42+dfsg-2ubuntu3.7 OpenLDAP server (slapd), procedure like the
below was used. Your mileage may vary.

```

$ scp admin@10.59.143.65:/usr/share/doc/sudo-ldap/schema.OpenLDAP /etc/ldap/schema/sudo-ldap.schema
admin@10.59.143.65's password:
schema.OpenLDAP                               100% 2410     2.4KB/s   00:00
$

$ touch /tmp/ldif/conf
$ echo 'include /etc/ldap/schema/sudo-ldap.schema' > /tmp/ldif/conf
$ slaptest -f /tmp/ldif/conf -F /tmp/ldif
config file testing succeeded
$

$ vi /tmp/ldif/cn\=config/cn\=schema/cn\=\{0}sudo-ldap.ldif

[ Remove the header and trailer lines so now it looks something like this ]

$ cat /tmp/ldif/cn\=config/cn\=schema/cn\=\{0}sudo-ldap.ldif
dn: cn=sudo-ldap,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: sudo-ldap
olcAttributeTypes: {0}( 1.3.6.1.4.1.15953.9.1.1 NAME 'sudoUser' DESC 'User(s
 ) who may  run sudo' EQUALITY caseExactIA5Match SUBSTR caseExactIA5Substrin
 gsMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: {1}( 1.3.6.1.4.1.15953.9.1.2 NAME 'sudoHost' DESC 'Host(s
 ) who may run sudo' EQUALITY caseExactIA5Match SUBSTR caseExactIA5Substring
 sMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: {2}( 1.3.6.1.4.1.15953.9.1.3 NAME 'sudoCommand' DESC 'Com
 mand(s) to be executed by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4
 .1.1466.115.121.1.26 )
olcAttributeTypes: {3}( 1.3.6.1.4.1.15953.9.1.4 NAME 'sudoRunAs' DESC 'User(
 s) impersonated by sudo (deprecated)' EQUALITY caseExactIA5Match SYNTAX 1.3
 .6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: {4}( 1.3.6.1.4.1.15953.9.1.5 NAME 'sudoOption' DESC 'Opti
 ons(s) followed by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466
 .115.121.1.26 )
olcAttributeTypes: {5}( 1.3.6.1.4.1.15953.9.1.6 NAME 'sudoRunAsUser' DESC 'U
 ser(s) impersonated by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.
 1466.115.121.1.26 )
olcAttributeTypes: {6}( 1.3.6.1.4.1.15953.9.1.7 NAME 'sudoRunAsGroup' DESC '
 Group(s) impersonated by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.
 1.1466.115.121.1.26 )
olcAttributeTypes: {7}( 1.3.6.1.4.1.15953.9.1.8 NAME 'sudoNotBefore' DESC 'S
 tart of time interval for which the entry is valid' EQUALITY generalizedTim
 eMatch ORDERING generalizedTimeOrderingMatch SYNTAX 1.3.6.1.4.1.1466.115.12
 1.1.24 )
olcAttributeTypes: {8}( 1.3.6.1.4.1.15953.9.1.9 NAME 'sudoNotAfter' DESC 'En
 d of time interval for which the entry is valid' EQUALITY generalizedTimeMa
 tch ORDERING generalizedTimeOrderingMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1
 .24 )
olcAttributeTypes: {9}( 1.3.6.1.4.1.15953.9.1.10 NAME 'sudoOrder' DESC 'an i
 nteger to order the sudoRole entries' EQUALITY integerMatch ORDERING intege
 rOrderingMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 )
olcObjectClasses: {0}( 1.3.6.1.4.1.15953.9.2.1 NAME 'sudoRole' DESC 'Sudoer 
 Entries' SUP top STRUCTURAL MUST cn MAY ( sudoUser $ sudoHost $ sudoCommand
  $ sudoRunAs $ sudoRunAsUser $ sudoRunAsGroup $ sudoOption $ sudoOrder $ su
 doNotBefore $ sudoNotAfter $ description ) )
$


$ cp /tmp/ldif/cn\=config/cn\=schema/cn\=\{0}sudo-ldap.ldif /etc/ldap/schema/sudo-ldap.ldif

$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/sudo-ldap.ldifSASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=sudo-ldap,cn=schema,cn=config"

$

```

### TACACS+ Server user

```
...
user = ldapuser {
        default service = permit
        chap = cleartext "ldapuser"
        pap = cleartext "ldapuser"
        service = exec {
        priv-lvl=15
        }
}
...

```

### RADIUS Server user

```
...
raduser1 Cleartext-Password := "password"
 Management-Privilege-Level = 1
...
ldapuser Cleartext-Password := ldapuser
 Brocade-Auth-Role = admin,
 Management-Privilege-Level = 15
...

```

## UT Authentication LDAP

### Config DB: Before

```

127.0.0.1:6379[4]> keys *LDAP*
(empty list or set)
127.0.0.1:6379[4]> keys *AAA*
(empty list or set)
127.0.0.1:6379[4]>

```

### Click: show aaa, show ldap: Before

```

admin@L10:~/show$ show aaa
AAA authentication login local (default)
AAA authentication failthrough False (default)
AAA nss passwd login (default)
AAA nss shadow login (default)
AAA nss group login (default)
AAA nss netgroup local (default)
AAA authorization login local (default)

admin@L10:~/show$

admin@L10:~/show$ show ldap
LDAP global timelimit 0 (default)
LDAP global retry 0 (default)
LDAP global ldap_version 3 (default)
LDAP global scope sub (default)
LDAP global idle_timelimit 0 (default)
LDAP global ssl off (default)
LDAP global use_type all (default)
LDAP global port 389 (default)
LDAP global bind_timelimit 10 (default)

admin@L10:~/show$

```

### Click CLI

```

sudo config ldap add 10.59.143.229
sudo config ldap base dc=sji,dc=broadcom,dc=net
sudo config aaa authentication login ldap local
sudo config aaa authentication failthrough enable

```

### KLISH: show aaa, show ldap-server: Before

```
sonic# show aaa
---------------------------------------------------------
AAA Authentication Information
---------------------------------------------------------
failthrough  : False
login-method : local
sonic#
sonic# show ldap-server
sonic#

```

### KLISH CLI

```

sonic(config)# ldap-server host 10.59.143.229
sonic(config)# ldap-server base dc=sji,dc=broadcom,dc=net
sonic(config)# aaa authentication failthrough enable
sonic(config)# aaa authentication login default group ldap local

```

### Config DB: After

```

127.0.0.1:6379[4]> keys *LDAP*
1) "LDAP|global"
2) "LDAP_SERVER|10.59.143.229"
127.0.0.1:6379[4]> keys *AAA*
1) "AAA|authentication"
127.0.0.1:6379[4]> hgetall LDAP|global
1) "base"
2) "dc=sji,dc=broadcom,dc=net"
127.0.0.1:6379[4]> hgetall LDAP_SERVER|10.59.143.229
1) "priority"
2) "1"
3) "retry"
4) "0"
5) "use_type"
6) "all"
127.0.0.1:6379[4]> hgetall AAA|authentication
1) "login"
2) "ldap,local"
3) "failthrough"
4) "True"
127.0.0.1:6379[4]>

```

### Click: show aaa, show ldap: After

```

admin@L10:~/show$ id ldapuser
uid=1201(ldapuser) gid=1000(admin) groups=1000(admin),27(sudo),999(docker)
admin@L10:~/show$ show ldap
LDAP global timelimit 0 (default)
LDAP global retry 0 (default)
LDAP global ldap_version 3 (default)
LDAP global scope sub (default)
LDAP global idle_timelimit 0 (default)
LDAP global ssl off (default)
LDAP global base dc=sji,dc=broadcom,dc=net
LDAP global use_type all (default)
LDAP global port 389 (default)
LDAP global bind_timelimit 10 (default)

LDAP_SERVER address 10.59.143.229
               priority 1
               retry 0
               use_type all

admin@L10:~/show$ show aaa
AAA authentication login ldap,local
AAA authentication failthrough True
AAA nss passwd login (default)
AAA nss shadow login (default)
AAA nss group login (default)
AAA nss netgroup local (default)
AAA authorization login local (default)

admin@L10:~/show$

```

### KLISH: show aaa, show ldap-server: After

```
sonic# show aaa
---------------------------------------------------------
AAA Authentication Information
---------------------------------------------------------
failthrough  : True
login-method : ldap, local
sonic#
sonic# show ldap-server
---------------------------------------------------------
LDAP  Global Configuration
---------------------------------------------------------
binddn        : dc=sji,dc=broadcom,dc=net
--------------------------------------------------------------------------------
HOST            USE-TYPE PRIORITY SSL       RETRY
--------------------------------------------------------------------------------
10.59.143.229   -         1       -         -
sonic#

```

###  Verify ldapuser can login with LDAP password.

```

$ ssh ldapuser@10.59.143.109
ldapuser@10.59.143.109's password: 
Creating directory '/home/ldapuser'.
Linux L10 4.9.0-11-2-amd64 #1 SMP Debian 4.9.189-3+deb9u2 (2019-11-11) x86_64
...
ldapuser@L10:~$ 

```

## UT Authentication TACPLUS with Name Service LDAP

### Click CLI

```

sudo config tacacs add 10.59.143.229
sudo config tacacs passkey testing123
sudo config aaa authentication failthrough enable
sudo config aaa authentication login tacacs+ local

sudo config ldap add 10.59.143.229
sudo config ldap base dc=sji,dc=broadcom,dc=net
sudo config aaa nss passwd ldap
sudo config aaa nss shadow ldap
sudo config aaa nss group ldap

```

### KLISH CLI

```

sonic(config)# tacacs-server host 10.59.143.229
sonic(config)# tacacs-server key testing123
sonic(config)# aaa authentication failthrough enable
sonic(config)# aaa authentication login default group tacacs+ local

sonic(config)# ldap-server host 10.59.143.229
sonic(config)# ldap-server base dc=sji,dc=broadcom,dc=net
sonic(config)# aaa name-service passwd group ldap
sonic(config)# aaa name-service shadow group ldap
sonic(config)# aaa name-service group group ldap

```

### Config DB: After

```

127.0.0.1:6379[4]> keys *AAA*
1) "AAA|nss"
2) "AAA|authentication"
127.0.0.1:6379[4]> keys *TACPLUS*
1) "TACPLUS_SERVER|10.59.143.229"
2) "TACPLUS|global"
127.0.0.1:6379[4]> keys *LDAP*
1) "LDAP|global"
2) "LDAP_SERVER|10.59.143.229"
127.0.0.1:6379[4]> hgetall AAA|nss
1) "passwd"
2) "ldap"
3) "shadow"
4) "ldap"
5) "group"
6) "ldap"
127.0.0.1:6379[4]> hgetall AAA|authentication
1) "failthrough"
2) "True"
3) "login"
4) "tacacs+,local"
127.0.0.1:6379[4]> hgetall TACPLUS|global
1) "passkey"
2) "testing123"
127.0.0.1:6379[4]> hgetall TACPLUS_SERVER|10.59.143.229
1) "priority"
2) "1"
3) "tcp_port"
4) "49"
127.0.0.1:6379[4]> hgetall LDAP|global
1) "base"
2) "dc=sji,dc=broadcom,dc=net"
127.0.0.1:6379[4]> hgetall LDAP_SERVER|10.59.143.229
1) "priority"
2) "1"
3) "retry"
4) "0"
5) "use_type"
6) "all"
127.0.0.1:6379[4]> 

```

### Click: show aaa, show tacacs, show ldap: After

```

admin@L10:~$ show aaa
AAA authentication login tacacs+,local
AAA authentication failthrough True
AAA nss group ldap
AAA nss passwd ldap
AAA nss shadow ldap
AAA nss netgroup local (default)
AAA authorization login local (default)

admin@L10:~$ show tacacs
TACPLUS global auth_type pap (default)
TACPLUS global timeout 5 (default)
TACPLUS global passkey testing123

TACPLUS_SERVER address 10.59.143.229
               priority 1
               tcp_port 49

admin@L10:~$ show ldap
LDAP global timelimit 0 (default)
LDAP global retry 0 (default)
LDAP global ldap_version 3 (default)
LDAP global scope sub (default)
LDAP global idle_timelimit 0 (default)
LDAP global ssl off (default)
LDAP global base dc=sji,dc=broadcom,dc=net
LDAP global use_type all (default)
LDAP global port 389 (default)
LDAP global bind_timelimit 10 (default)

LDAP_SERVER address 10.59.143.229
               priority 1
               retry 0
               use_type all

admin@L10:~$ 

```

### KLISH: show aaa, show tacacs-server, show ldap-server: After

```
sonic# show aaa
---------------------------------------------------------
AAA Authentication Information
---------------------------------------------------------
failthrough  : True
login-method : tacacs+, local
---------------------------------------------------------
AAA Name-Service Information
---------------------------------------------------------
passwd  : ldap
shadow  : ldap
group   : ldap
sonic#
sonic# show tacacs-server global
---------------------------------------------------------
TACACS Global Configuration
---------------------------------------------------------
key        : testing123
sonic# 
sonic# show tacacs-server host
--------------------------------------------------------------------------------
HOST                 AUTH-TYPE       KEY        PORT       PRIORITY   TIMEOUT   
--------------------------------------------------------------------------------
10.59.143.229        pap                        49         1          5         
sonic#
sonic# show ldap-server
---------------------------------------------------------
LDAP  Global Configuration
---------------------------------------------------------
binddn        : dc=sji,dc=broadcom,dc=net
--------------------------------------------------------------------------------
HOST            USE-TYPE PRIORITY SSL       RETRY
--------------------------------------------------------------------------------
10.59.143.229   -         1       -         -
sonic#

```

### Verify user can login using TACACS+ password with LDAP Name Service

```

$ ssh ldapuser@10.59.143.109
ldapuser@10.59.143.109's password: 
Creating directory '/home/ldapuser'.
Linux L10 4.9.0-11-2-amd64 #1 SMP Debian 4.9.189-3+deb9u2 (2019-11-11) x86_64
...
ldapuser@L10:~$ 

```

## UT Authentication RADIUS with Name Service LDAP

### Click CLI

```

sudo config radius add 10.59.143.229
sudo config radius passkey sharedsecret
sudo config aaa authentication failthrough enable
sudo config aaa authentication login radius local

sudo config ldap add 10.59.143.229
sudo config ldap base dc=sji,dc=broadcom,dc=net
sudo config aaa nss passwd ldap
sudo config aaa nss shadow ldap
sudo config aaa nss group ldap

```

### KLISH CLI

```

sonic(config)# radius-server host 10.59.143.229
sonic(config)# radius-server key sharedsecret
sonic(config)# aaa authentication failthrough enable
sonic(config)# aaa authentication login default group radius local

sonic(config)# ldap-server host 10.59.143.229
sonic(config)# ldap-server base dc=sji,dc=broadcom,dc=net
sonic(config)# aaa name-service passwd group ldap
sonic(config)# aaa name-service shadow group ldap
sonic(config)# aaa name-service group group ldap

```

### Config DB: After

```

127.0.0.1:6379[4]> keys *AAA*
1) "AAA|nss"
2) "AAA|authentication"
127.0.0.1:6379[4]> keys *RADIUS*
1) "RADIUS_SERVER|10.59.143.229"
2) "RADIUS|global"
127.0.0.1:6379[4]> keys *LDAP*
1) "LDAP|global"
2) "LDAP_SERVER|10.59.143.229"
127.0.0.1:6379[4]> hgetall AAA|nss
1) "passwd"
2) "ldap"
3) "shadow"
4) "ldap"
5) "group"
6) "ldap"
127.0.0.1:6379[4]> hgetall AAA|authentication
1) "failthrough"
2) "True"
3) "login"
4) "radius,local"
127.0.0.1:6379[4]> hgetall RADIUS|global
1) "passkey"
2) "sharedsecret"
127.0.0.1:6379[4]> hgetall RADIUS_SERVER|10.59.143.229
1) "priority"
2) "1"
3) "auth_port"
4) "1812"
127.0.0.1:6379[4]> hgetall LDAP|global
1) "base"
2) "dc=sji,dc=broadcom,dc=net"
127.0.0.1:6379[4]> hgetall LDAP_SERVER|10.59.143.229
1) "priority"
2) "1"
3) "retry"
4) "0"
5) "use_type"
6) "all"
127.0.0.1:6379[4]> 

```

### show aaa, show radius, show ldap: After

```

admin@L10:~$ show aaa
AAA authentication login radius,local
AAA authentication failthrough True
AAA nss group ldap
AAA nss passwd ldap
AAA nss shadow ldap
AAA nss netgroup local (default)
AAA authorization login local (default)

admin@L10:~$ show radius
RADIUS global auth_type pap (default)
RADIUS global retransmit 3 (default)
RADIUS global timeout 5 (default)
RADIUS global passkey sharedsecret

RADIUS_SERVER address 10.59.143.229
               priority 1
               auth_port 1812

admin@L10:~$

admin@L10:~$ show ldap
LDAP global timelimit 0 (default)
LDAP global retry 0 (default)
LDAP global ldap_version 3 (default)
LDAP global scope sub (default)
LDAP global idle_timelimit 0 (default)
LDAP global ssl off (default)
LDAP global base dc=sji,dc=broadcom,dc=net
LDAP global use_type all (default)
LDAP global port 389 (default)
LDAP global bind_timelimit 10 (default)

LDAP_SERVER address 10.59.143.229
               priority 1
               retry 0
               use_type all

admin@L10:~$ 

```

### KLISH: show aaa, show radius-server, show ldap-server: After

```
sonic# show aaa
---------------------------------------------------------
AAA Authentication Information
---------------------------------------------------------
failthrough  : True
login-method : radius, local
---------------------------------------------------------
AAA Name-Service Information
---------------------------------------------------------
passwd  : ldap
shadow  : ldap
group   : ldap
sonic#
sonic# show radius-server
---------------------------------------------------------
RADIUS Global Configuration
---------------------------------------------------------
key        : sharedsecret
--------------------------------------------------------------------------------
HOST            AUTH-TYPE KEY       AUTH-PORT PRIORITY TIMEOUT RTSMT VRF  
--------------------------------------------------------------------------------
10.59.143.229   -         -         1812      1        -       -     -    
sonic#
sonic# show ldap-server
---------------------------------------------------------
LDAP  Global Configuration
---------------------------------------------------------
binddn        : dc=sji,dc=broadcom,dc=net
--------------------------------------------------------------------------------
HOST            USE-TYPE PRIORITY SSL       RETRY
--------------------------------------------------------------------------------
10.59.143.229   -         1       -         -
sonic#

```

### Verify user can login using RADIUS password with LDAP Name Service

```

$ ssh ldapuser@10.59.143.109
ldapuser@10.59.143.109's password: 
Creating directory '/home/ldapuser'.
Linux L10 4.9.0-11-2-amd64 #1 SMP Debian 4.9.189-3+deb9u2 (2019-11-11) x86_64
...
ldapuser@L10:~$ 

```

## UT Authentication RADIUS with Logon Authorization LDAP and Name Service LDAP

### Click CLI

```

sudo config radius add 10.59.143.229
sudo config radius passkey sharedsecret
sudo config aaa authentication failthrough enable
sudo config aaa authentication login radius local

sudo config ldap add 10.59.143.229
sudo config ldap base dc=sji,dc=broadcom,dc=net
sudo config aaa nss passwd ldap
sudo config aaa nss shadow ldap
sudo config aaa nss group ldap

sudo config ldap pam_groupdn cn=sudo,ou=Group,dc=sji,dc=broadcom,dc=net
sudo config ldap pam_member_attribute memberUid
sudo config aaa authorization login ldap

```

### KLISH CLI

```

sonic(config)# radius-server host 10.59.143.229
sonic(config)# radius-server key sharedsecret
sonic(config)# aaa authentication failthrough enable
sonic(config)# aaa authentication login default group radius local

sonic(config)# ldap-server host 10.59.143.229
sonic(config)# ldap-server base dc=sji,dc=broadcom,dc=net
sonic(config)# aaa name-service passwd group ldap
sonic(config)# aaa name-service shadow group ldap
sonic(config)# aaa name-service group group ldap

sonic(config)# ldap-server pam-groupdn cn=sudo,ou=Group,dc=sji,dc=broadcom,dc=net
sonic(config)# ldap-server pam-member-attribute memberUid
sonic(config)# aaa authorization login default group ldap

```

### Config DB: After

```

127.0.0.1:6379[4]> keys *AAA*
1) "AAA|authorization"
2) "AAA|nss"
3) "AAA|authentication"
127.0.0.1:6379[4]> keys *RADIUS*
1) "RADIUS_SERVER|10.59.143.229"
2) "RADIUS|global"
127.0.0.1:6379[4]> keys *LDAP*
1) "LDAP|global"
2) "LDAP_SERVER|10.59.143.229"
127.0.0.1:6379[4]> hgetall AAA|nss
1) "passwd"
2) "ldap"
3) "shadow"
4) "ldap"
5) "group"
6) "ldap"
127.0.0.1:6379[4]> hgetall AAA|authentication
1) "failthrough"
2) "True"
3) "login"
4) "radius,local"
127.0.0.1:6379[4]> hgetall AAA|authorization
1) "login"
2) "ldap"
127.0.0.1:6379[4]> hgetall RADIUS|global
1) "passkey"
2) "sharedsecret"
127.0.0.1:6379[4]> hgetall RADIUS_SERVER|10.59.143.229
1) "priority"
2) "1"
3) "auth_port"
4) "1812"
127.0.0.1:6379[4]> hgetall LDAP|global
1) "base"
2) "dc=sji,dc=broadcom,dc=net"
3) "pam_groupdn"
4) "cn=sudo,ou=Group,dc=sji,dc=broadcom,dc=net"
5) "pam_member_attribute"
6) "memberUid"
127.0.0.1:6379[4]>

```

### show aaa, show radius, show ldap: After

```

admin@L10:~$ show aaa
AAA authentication login radius,local
AAA authentication failthrough True
AAA nss group ldap
AAA nss passwd ldap
AAA nss shadow ldap
AAA nss netgroup local (default)
AAA authorization login ldap

admin@L10:~$

admin@L10:~$ show radius
RADIUS global auth_type pap (default)
RADIUS global retransmit 3 (default)
RADIUS global timeout 5 (default)
RADIUS global passkey sharedsecret

RADIUS_SERVER address 10.59.143.229
               priority 1
               auth_port 1812

admin@L10:~$

admin@L10:~$ show ldap
LDAP global retry 0 (default)
LDAP global ssl off (default)
LDAP global idle_timelimit 0 (default)
LDAP global bind_timelimit 10 (default)
LDAP global timelimit 0 (default)
LDAP global pam_member_attribute memberUid
LDAP global pam_groupdn cn=sudo,ou=Group,dc=sji,dc=broadcom,dc=net
LDAP global ldap_version 3 (default)
LDAP global scope sub (default)
LDAP global base dc=sji,dc=broadcom,dc=net
LDAP global use_type all (default)
LDAP global port 389 (default)

LDAP_SERVER address 10.59.143.229
               priority 1
               retry 0
               use_type all

admin@L10:~$ 

```

### KLISH: show aaa, show radius-server, show ldap-server: After

```
sonic# show aaa
---------------------------------------------------------
AAA Authentication Information
---------------------------------------------------------
failthrough  : True
login-method : radius, local
---------------------------------------------------------
AAA Authorization Information
---------------------------------------------------------
login        : ldap
---------------------------------------------------------
AAA Name-Service Information
---------------------------------------------------------
passwd  : ldap
shadow  : ldap
group   : ldap
sonic#
sonic# show radius-server
---------------------------------------------------------
RADIUS Global Configuration
---------------------------------------------------------
key        : sharedsecret
--------------------------------------------------------------------------------
HOST            AUTH-TYPE KEY       AUTH-PORT PRIORITY TIMEOUT RTSMT VRF  
--------------------------------------------------------------------------------
10.59.143.229   -         -         1812      1        -       -     -    
sonic#
sonic# show ldap-server
---------------------------------------------------------
LDAP  Global Configuration
---------------------------------------------------------
binddn               : dc=sji,dc=broadcom,dc=net
pam_member_attribute : memberUid
pam_groupdn          : cn=sudo,ou=Group,dc=sji,dc=broadcom,dc=net
--------------------------------------------------------------------------------
HOST            USE-TYPE PRIORITY SSL       RETRY
--------------------------------------------------------------------------------
10.59.143.229   -         1       -         -
sonic#

```

### Verify user can login using RADIUS password with LDAP Name Service

```

$ ssh ldapuser@10.59.143.109
ldapuser@10.59.143.109's password: 
Creating directory '/home/ldapuser'.
Linux L10 4.9.0-11-2-amd64 #1 SMP Debian 4.9.189-3+deb9u2 (2019-11-11) x86_64
...
ldapuser@L10:~$ 

```


## UT Authentication RADIUS with Name Service LDAP and sudo Authorization LDAP

### Click CLI

```

sudo config radius add 10.59.143.229
sudo config radius passkey sharedsecret
sudo config aaa authentication failthrough enable
sudo config aaa authentication login radius local

sudo config ldap add 10.59.143.229
sudo config ldap base dc=sji,dc=broadcom,dc=net
sudo config aaa nss passwd ldap
sudo config aaa nss shadow ldap
sudo config aaa nss group ldap

sudo config aaa nss sudoers ldap
sudo config ldap sudoers_base ou=Sudoers,dc=sji,dc=broadcom,dc=net


```

### KLISH CLI

```

sonic(config)# radius-server host 10.59.143.229
sonic(config)# radius-server key sharedsecret
sonic(config)# aaa authentication failthrough enable
sonic(config)# aaa authentication login default group radius local

sonic(config)# ldap-server host 10.59.143.229
sonic(config)# ldap-server base dc=sji,dc=broadcom,dc=net
sonic(config)# aaa name-service passwd group ldap
sonic(config)# aaa name-service shadow group ldap
sonic(config)# aaa name-service group group ldap

sonic(config)# aaa name-service sudoers group ldap
sonic(config)# ldap-server sudoers-base ou=Sudoers,dc=sji,dc=broadcom,dc=net

```

### Config DB: After

```

127.0.0.1:6379[4]> keys *AAA*
1) "AAA|nss"
2) "AAA|authentication"
127.0.0.1:6379[4]> keys *RADIUS*
1) "RADIUS_SERVER|10.59.143.229"
2) "RADIUS|global"
127.0.0.1:6379[4]> keys *LDAP*
1) "LDAP|global"
2) "LDAP_SERVER|10.59.143.229"
127.0.0.1:6379[4]> hgetall AAA|nss
1) "passwd"
2) "ldap"
3) "shadow"
4) "ldap"
5) "group"
6) "ldap"
7) "sudoers"
8) "ldap"
127.0.0.1:6379[4]> hgetall AAA|authentication
1) "failthrough"
2) "True"
3) "login"
4) "radius,local"
127.0.0.1:6379[4]> hgetall RADIUS|global
1) "passkey"
2) "sharedsecret"
127.0.0.1:6379[4]> hgetall RADIUS_SERVER|10.59.143.229
1) "priority"
2) "1"
3) "auth_port"
4) "1812"
127.0.0.1:6379[4]> hgetall LDAP|global
1) "base"
2) "dc=sji,dc=broadcom,dc=net"
3) "sudoers_base"
4) "ou=Sudoers,dc=sji,dc=broadcom,dc=net"
127.0.0.1:6379[4]> hgetall LDAP_SERVER|10.59.143.229
1) "priority"
2) "1"
3) "retry"
4) "0"
5) "use_type"
6) "all"
127.0.0.1:6379[4]> 

```

### show aaa, show radius, show ldap: After

```

admin@L10:~$ show aaa
AAA authentication login radius,local
AAA authentication failthrough True
AAA nss group ldap
AAA nss passwd ldap
AAA nss sudoers ldap
AAA nss netgroup local (default)
AAA nss shadow ldap
AAA authorization login local (default)

admin@L10:~$

admin@L10:~$ show radius
RADIUS global auth_type pap (default)
RADIUS global retransmit 3 (default)
RADIUS global timeout 5 (default)
RADIUS global passkey sharedsecret

RADIUS_SERVER address 10.59.143.229
               priority 1
               auth_port 1812

admin@L10:~$

admin@L10:~$ show ldap
LDAP global retry 0 (default)
LDAP global ssl off (default)
LDAP global sudoers_base ou=Sudoers,dc=sji,dc=broadcom,dc=net
LDAP global idle_timelimit 0 (default)
LDAP global bind_timelimit 10 (default)
LDAP global timelimit 0 (default)
LDAP global ldap_version 3 (default)
LDAP global scope sub (default)
LDAP global base dc=sji,dc=broadcom,dc=net
LDAP global use_type all (default)
LDAP global port 389 (default)

LDAP_SERVER address 10.59.143.229
               priority 1
               retry 0
               use_type all

admin@L10:~$

```

### KLISH: show aaa, show radius-server, show ldap-server: After

```
sonic# show aaa
---------------------------------------------------------
AAA Authentication Information
---------------------------------------------------------
failthrough  : True
login-method : radius, local
---------------------------------------------------------
AAA Name-Service Information
---------------------------------------------------------
passwd  : ldap
shadow  : ldap
group   : ldap
sudoers : ldap
sonic#
sonic# show radius-server
---------------------------------------------------------
RADIUS Global Configuration
---------------------------------------------------------
key        : sharedsecret
--------------------------------------------------------------------------------
HOST            AUTH-TYPE KEY       AUTH-PORT PRIORITY TIMEOUT RTSMT VRF  
--------------------------------------------------------------------------------
10.59.143.229   -         -         1812      1        -       -     -    
sonic#
sonic# show ldap-server
---------------------------------------------------------
LDAP  Global Configuration
---------------------------------------------------------
binddn        : dc=sji,dc=broadcom,dc=net
sudoers-base  : ou=Sudoers,dc=sji,dc=broadcom,dc=net
--------------------------------------------------------------------------------
HOST            USE-TYPE PRIORITY SSL       RETRY
--------------------------------------------------------------------------------
10.59.143.229   -         1       -         -
sonic#

```

### Verify user can login using RADIUS password, LDAP Name Service, and sudo

```

$ ssh raduser1@10.59.143.109
raduser1@10.59.143.109's password: 
Linux L10 4.9.0-11-2-amd64 #1 SMP Debian 4.9.189-3+deb9u2 (2019-11-11) x86_64
...
raduser1@L10:~$ id
uid=1202(raduser1) gid=100(users) groups=100(users),999(docker)
raduser1@L10:~$ sudo -i
[sudo] password for raduser1: 
root@L10:~# id
uid=0(root) gid=0(root) groups=0(root)
root@L10:~# exit
logout
raduser1@L10:~$

```

## UT Authentication Non Global NSS PAM SUDO options

### Click CLI

```

sudo config radius add 10.59.143.229
sudo config radius passkey sharedsecret
sudo config aaa authentication failthrough enable
sudo config aaa authentication login radius local

sudo config ldap add 10.59.143.229
sudo config ldap base dc=sji,dc=broadcom,dc=net

sudo config ldap nss base dc=sji,dc=broadcom,dc=net
sudo config aaa nss passwd ldap
sudo config aaa nss shadow ldap
sudo config aaa nss group ldap

sudo config aaa nss sudoers ldap
sudo config ldap sudo sudoers_base ou=Sudoers,dc=sji,dc=broadcom,dc=net

sudo config ldap pam pam_groupdn cn=docker,ou=Group,dc=sji,dc=broadcom,dc=net
sudo config ldap pam pam_member_attribute memberUid
sudo config aaa authorization login ldap

```

### KLISH CLI

```

sonic(config)# radius-server host 10.59.143.229
sonic(config)# radius-server key sharedsecret
sonic(config)# aaa authentication failthrough enable
sonic(config)# aaa authentication login default group radius local

sonic(config)# ldap-server host 10.59.143.229
sonic(config)# ldap-server base dc=sji,dc=broadcom,dc=net
sonic(config)# ldap-server nss base dc=sji,dc=broadcom,dc=net
sonic(config)# aaa name-service passwd group ldap
sonic(config)# aaa name-service shadow group ldap
sonic(config)# aaa name-service group group ldap

sonic(config)# aaa name-service sudoers group ldap
sonic(config)# ldap-server sudo sudoers-base ou=Sudoers,dc=sji,dc=broadcom,dc=net

sonic(config)# ldap-server pam pam-groupdn cn=docker,ou=Group,dc=sji,dc=broadcom,dc=net
sonic(config)# ldap-server pam pam-member-attribute memberUid
sonic(config)# aaa authorization login default group ldap

```

### Config DB: After

```

127.0.0.1:6379[4]> keys *AAA*
1) "AAA|authorization"
2) "AAA|nss"
3) "AAA|authentication"
127.0.0.1:6379[4]> keys *RADIUS*
1) "RADIUS_SERVER|10.59.143.229"
2) "RADIUS|global"
127.0.0.1:6379[4]> keys *LDAP*
1) "LDAP|global"
2) "LDAP|nss"
3) "LDAP|pam"
4) "LDAP|sudo"
5) "LDAP_SERVER|10.59.143.229"
127.0.0.1:6379[4]> hgetall AAA|nss
1) "passwd"
2) "ldap"
3) "shadow"
4) "ldap"
5) "group"
6) "ldap"
5) "sudoers"
6) "ldap"
127.0.0.1:6379[4]> hgetall AAA|authentication
1) "failthrough"
2) "True"
3) "login"
4) "radius,local"
127.0.0.1:6379[4]> hgetall AAA|authorization
1) "login"
2) "ldap"
127.0.0.1:6379[4]> hgetall RADIUS|global
1) "passkey"
2) "sharedsecret"
127.0.0.1:6379[4]> hgetall RADIUS_SERVER|10.59.143.229
1) "priority"
2) "1"
3) "auth_port"
4) "1812"
127.0.0.1:6379[4]> hgetall LDAP|global
1) "base"
2) "dc=sji,dc=broadcom,dc=net"
127.0.0.1:6379[4]> hgetall LDAP|nss
1) "base"
2) "dc=sji,dc=broadcom,dc=net"
127.0.0.1:6379[4]> hgetall LDAP|pam
1) "pam_groupdn"
2) "cn=docker,ou=Group,dc=sji,dc=broadcom,dc=net"
3) "pam_member_attribute"
4) "memberUid"
127.0.0.1:6379[4]> hgetall LDAP|sudo
1) "sudoers_base"
2) "ou=Sudoers,dc=sji,dc=broadcom,dc=net"
127.0.0.1:6379[4]>

```

### show aaa, show radius, show ldap: After

```

admin@L10:~$ show aaa
AAA authentication login radius,local
AAA authentication failthrough True
AAA nss group ldap
AAA nss passwd ldap
AAA nss shadow ldap
AAA nss sudoers ldap
AAA nss netgroup local (default)
AAA authorization login ldap

admin@L10:~$

admin@L10:~$ show radius
RADIUS global auth_type pap (default)
RADIUS global retransmit 3 (default)
RADIUS global timeout 5 (default)
RADIUS global passkey sharedsecret

RADIUS_SERVER address 10.59.143.229
               priority 1
               auth_port 1812

admin@L10:~$

admin@L10:~$ show ldap
LDAP global retry 0 (default)
LDAP global ssl off (default)
LDAP global idle_timelimit 0 (default)
LDAP global bind_timelimit 10 (default)
LDAP global timelimit 0 (default)
LDAP global ldap_version 3 (default)
LDAP global scope sub (default)
LDAP global base dc=sji,dc=broadcom,dc=net
LDAP global use_type all (default)
LDAP global port 389 (default)
LDAP nss base dc=sji,dc=broadcom,dc=net
LDAP pam pam_member_attribute memberUid
LDAP pam pam_groupdn cn=docker,ou=Group,dc=sji,dc=broadcom,dc=net
LDAP sudo sudoers_base ou=Sudoers,dc=sji,dc=broadcom,dc=net

LDAP_SERVER address 10.59.143.229
               priority 1
               retry 0
               use_type all

admin@L10:~$ 

```

### KLISH: show aaa, show radius-server, show ldap-server: After

```
sonic# show aaa
---------------------------------------------------------
AAA Authentication Information
---------------------------------------------------------
failthrough  : True
login-method : radius, local
---------------------------------------------------------
AAA Authorization Information
---------------------------------------------------------
login        : ldap
---------------------------------------------------------
AAA Name-Service Information
---------------------------------------------------------
passwd  : ldap
shadow  : ldap
group   : ldap
sudoers : ldap
sonic#
sonic# show radius-server
---------------------------------------------------------
RADIUS Global Configuration
---------------------------------------------------------
key        : sharedsecret
--------------------------------------------------------------------------------
HOST            AUTH-TYPE KEY       AUTH-PORT PRIORITY TIMEOUT RTSMT VRF  
--------------------------------------------------------------------------------
10.59.143.229   -         -         1812      1        -       -     -    
sonic#
sonic# show ldap-server
---------------------------------------------------------
LDAP  Global Configuration
---------------------------------------------------------
binddn               : dc=sji,dc=broadcom,dc=net
---------------------------------------------------------
LDAP  NSS Configuration
---------------------------------------------------------
binddn               : dc=sji,dc=broadcom,dc=net
---------------------------------------------------------
LDAP  PAM Configuration
---------------------------------------------------------
pam_member_attribute : memberUid
pam_groupdn          : cn=docker,ou=Group,dc=sji,dc=broadcom,dc=net
---------------------------------------------------------
LDAP  sudo Configuration
---------------------------------------------------------
sudoers_base         : ou=Sudoers,dc=sji,dc=broadcom,dc=net
--------------------------------------------------------------------------------
HOST            USE-TYPE PRIORITY SSL       RETRY
--------------------------------------------------------------------------------
10.59.143.229   -         1       -         -
sonic#

```

### Verify user can login using RADIUS password with LDAP Name Service LDAP logon authorization and sudo authorization

```

$ ssh raduser1@10.59.143.109
raduser1@10.59.143.109's password: 
Linux L10 4.9.0-11-2-amd64 #1 SMP Debian 4.9.189-3+deb9u2 (2019-11-11) x86_64
...
raduser1@L10:~$ id
uid=1202(raduser1) gid=100(users) groups=100(users),999(docker)
raduser1@L10:~$ sudo -i
[sudo] password for raduser1: 
root@L10:~# id
uid=0(root) gid=0(root) groups=0(root)
root@L10:~# exit
logout
raduser1@L10:~$

```

## UT Authentication LDAP AD ? For Implementation In a Future Release (IIFR)

### Click CLI

```

sudo config ldap add 10.75.16.174
sudo config ldap base 'OU=San Jose,OU=California,OU=US,OU=Users,OU=Accounts,DC=Broadcom,DC=net'
sudo config ldap binddn 'binddn CN=Search(035)Account,OU=San Jose,OU=California,OU=US,OU=Users,OU=Accounts,DC=Broadcom,DC=net'
sudo config ldap bindpw searchPW
sudo config ldap pam_login_attribute sAMAccountName
sudo config ldap pam_filter objectclass=User

sudo config ldap map add attribute uid sAMAccountName shadowLastChange pwdLastSet uniqueMember member
sudo config ldap map add objectclass posixAccount user shadowAccount user posixGroup group


sudo config aaa authentication login ldap local
sudo config aaa authentication failthrough enable

```

### Config DB: After

```

127.0.0.1:6379[4]> keys *LDAP*
1) "LDAP_MAP|OBJECTCLASS|posixGroup"
2) "LDAP_MAP|ATTRIBUTE|shadowLastChange"
3) "LDAP_MAP|ATTRIBUTE|uid"
4) "LDAP_MAP|ATTRIBUTE|uniqueMember"
5) "LDAP_MAP|OBJECTCLASS|posixAccount"
6) "LDAP_MAP|OBJECTCLASS|shadowAccount"
7) "LDAP_SERVER|10.75.16.174"
8) "LDAP|global"
127.0.0.1:6379[4]> keys *AAA*
1) "AAA|authentication"
127.0.0.1:6379[4]> hgetall LDAP_SERVER|10.75.16.174
1) "priority"
2) "1"
3) "retry"
4) "0"
5) "use_type"
6) "all"
127.0.0.1:6379[4]> hgetall LDAP|global
 1) "pam_login_attribute"
 2) "sAMAccountName"
 3) "pam_filter"
 4) "objectclass=User"
 5) "binddn"
 6) "binddn CN=Search(035)Account,OU=San Jose,OU=California,OU=US,OU=Users,OU=Accounts,DC=Broadcom,DC=net"
 7) "bindpw"
 8) "searchPW"
 9) "base"
10) "OU=San Jose,OU=California,OU=US,OU=Users,OU=Accounts,DC=Broadcom,DC=net"
127.0.0.1:6379[4]> hgetall LDAP_MAP|OBJECTCLASS|posixGroup
1) "to"
2) "group"
127.0.0.1:6379[4]> hgetall LDAP_MAP|ATTRIBUTE|shadowLastChange
1) "to"
2) "pwdLastSet"
127.0.0.1:6379[4]> hgetall LDAP_MAP|ATTRIBUTE|uid
1) "to"
2) "sAMAccountName"
127.0.0.1:6379[4]> hgetall LDAP_MAP|ATTRIBUTE|uniqueMember
1) "to"
2) "member"
127.0.0.1:6379[4]> hgetall LDAP_MAP|OBJECTCLASS|posixAccount
1) "to"
2) "user"
127.0.0.1:6379[4]> hgetall LDAP_MAP|OBJECTCLASS|shadowAccount
1) "to"
2) "user"
127.0.0.1:6379[4]> 

```

### show aaa, show ldap: After

```

admin@L10:~$ id aduser
uid=95431(aduser) gid=95431 groups=95431
admin@L10:~$ 
admin@L10:~$ show aaa
AAA authentication login ldap,local
AAA authentication failthrough True
AAA nss passwd login (default)
AAA nss shadow login (default)
AAA nss group login (default)
AAA nss netgroup local (default)
AAA authorization login local (default)

admin@L10:~$ show ldap
LDAP global binddn CN=Search(035)Account,OU=San Jose,OU=California,OU=US,OU=Users,OU=Accounts,DC=Broadcom,DC=net
LDAP global retry 0 (default)
LDAP global ssl off (default)
LDAP global idle_timelimit 0 (default)
LDAP global bind_timelimit 10 (default)
LDAP global timelimit 0 (default)
LDAP global base OU=San Jose,OU=California,OU=US,OU=Users,OU=Accounts,DC=Broadcom,DC=net
LDAP global bindpw searchPW
LDAP global ldap_version 3 (default)
LDAP global pam_filter objectclass=User
LDAP global scope sub (default)
LDAP global pam_login_attribute sAMAccountName
LDAP global use_type all (default)
LDAP global port 389 (default)

LDAP_MAP_ATTRIBUTE
               uniqueMember member
               shadowLastChange pwdLastSet
               uid sAMAccountName

LDAP_MAP_OBJECTCLASS
               shadowAccount user
               posixGroup group
               posixAccount user

LDAP_SERVER address 10.75.16.174
               priority 1
               retry 0
               use_type all

admin@L10:~$ 

```

###  Verify AD user can login with LDAP password.

```
$ ssh aduser@10.59.143.205
aduser@10.59.143.205's password: 
Creating directory '/home/aduser'.
Linux L10 4.9.0-11-2-amd64 #1 SMP Debian 4.9.189-3+deb9u2 (2019-11-11) x86_64
...
aduser@L10:~$

```

# References

## nss_ldap

https://manpages.debian.org/testing/libnss-ldap/libnss-ldap.conf.5.en.html

## pam_ldap

https://manpages.debian.org/testing/libpam-ldapd/pam_ldap.8.en.html

## sudo_ldap

https://www.sudo.ws/man/1.8.17/sudoers.ldap.man.html

## Linux PAM

http://man7.org/linux/man-pages/man3/pam.3.html

## NSS Reference

http://man7.org/linux/man-pages/man5/nsswitch.conf.5.html

## LDAP client configuration file

https://linux.die.net/man/5/ldap.conf

## TACPLUS Authentication

https://github.com/Azure/SONiC/blob/master/doc/aaa/TACACS%2B%20Authentication.md

## RADIUS Management User Authentication

https://github.com/Azure/SONiC/blob/c82c8b67c31a63b1b438a143e2ce1a92b2b580fb/doc/aaa/radius_authentication.md


