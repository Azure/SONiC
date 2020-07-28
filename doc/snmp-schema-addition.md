# SONiC SNMP TABLE schema proposal #

## Scope of the change ##

Currently SNMP configuration is managed from a mix of yaml files and DB based ACLs, we propose to integrate the SMP configs into an SNMP table in the DB.  
The following document proposes the database Schema in Json format and the list of required changes to the code.

## Current configurtions for SNMP ##
sonic-buidimage and sonic-snmpagent depend on the following files:
1. */etc/sonic/snmp.yaml*
    defines 2 keys:  
        *snmp_rocommunity:*  
        *snmp_location:*  
   this file is consumed in the snmp docker container by */usr/share/sonic/templates/snmpd.conf.j2* and */usr/bin/snmpd-config-updater*.  
2. */etc/sonic/config_db.json* or */etc/sonic/minigraph.xml* for ACL definitions 'SNMP_ACL'
   this file is consumed by */usr/bin/snmpd-config-updater*, */usr/share/sonic/templates/alias_map.j2*, */usr/share/sonic/templates/sysDescription.j2*.

In the end this is used to produce */etc/snmp/snmpd.conf*.

## Limits and incentives to change ##
- This goes against the principle of having the config DB be the central repository of configuration data.
- Imposes the provisioning of multiple static files.
- the snmpContact is hardcoded in the */usr/share/sonic/templates/snmpd.conf.j2* file.
- pass-throughs are hard-coded.

## Proposal: integrate the SNMP configuration into config DB ##
### Proposed Schema ###

```
"SNMP": {
    "location": LOCATION_STRING,
    "contact": CONTACT_STRING,
    "v2c": {
        COMMUNITY_STRING: {
            "type": "rw"|"ro",
        }
    },
    "pass_trough": {
        OID_STRING: COMMAND_STRING,
    }
}
```

Where:
- LOCATION_STRING:  String, defines the snmpLocation, default: "".  
- CONTACT_STRING:   String, defines the snmpContact, default: "" or the current hardcoded value for backwards compatibility.  
- COMMUNITY_STRING: String, defines the community string.  
- OID_STRING:       String. a dotted notation OID prefix.  
- COMMAND_STRING:   String: command to be called when OID_STRING is requested.

New keys:
- "v2c": we define a "v2c" tree to allow for future expansion for other versions of the SNMP protocol, this spec only defines for SNMP v2  
         we could imagine the implementation of "v3" with the inclusion of users or references to central PAM methods.  
- "type":  Optional, if ommited defaults to 'ro', there are 2 possible values:  
           "ro": read-only, the only implemented method at this time.  
           "rw": well you never know - here for completeness but unused in the code.  
- "pass_trough": used for pass-trough definitions - currently the config only has one hard-coded pass-through defined for SysDescription.

### Files needing modification for implementation ###

The changes we propose are only additive to remain compatible with the current install base and the current way of doing things.

In repo *sonic-buidlimage*:

*dockers/docker-snmp-v2/snmpd.conf.j2*:  
    verify the existence of the SNMP table in the datatbase and fork behavior if present, if not continue using old method.

*dockers/docker-snmp-v2/snmpd-config-updater*:  
    this file will be deprecated soon by caclmgrd so no updates will be done


In repo *sonic-swss-common*: 

*common/schema.h*:  
```
#define CFG_SNMP_TABLE_NAME           "SNMP"
```

In repo *sonic-swss*:

*doc/swss-schema.md*:
    add the definition of this schema

## Unsolved Issues ##
- Uploading custom pass-through
