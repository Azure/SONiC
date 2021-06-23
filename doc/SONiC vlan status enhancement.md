# SONiC vlan status enhancement

## Catalogue

- **1. Overview**
- **2. Terminology**
- **3. Vlanmgrd introduction**
- **4. Problem Description**
- **5. Target**
- **6. High-Level Design**
- **7. Detailed Design**
- **8. Unit testing**

## 1. Overview

This document points out the problem of VLAN status in SONiC and puts forward a method to solve it.

## 2. Terminology

- **admin_status**：Configuration status of vlan-interface. Set by users through CLI command or config.json. Default value is “up”. This value is stored in CONFIG_DB. Vlan’s field of admin_status in config_db.json will not change.
- **member_status**: This value is stored in APPL_DB, it's new status representing the member ports' operating status of vlan-interface. Value is “up” when at least one of the member ports’s oper_status is “up”, and “down” when one of the following conditions is matched:
  - All member ports' oper_status are “down”. 
  - No member port in this vlan-interface.
    
    
- **oper_status**: This value is stored in APPL_DB, it's new status representing the operating status of the vlan-interface. It’s also the status of Vlanxx in Linux kernel, set via Linux command “ip link set Vlanxx <up|down>”. 

## 3. Vlanmgrd introduction

Vlanmgrd implements the functions such as adding, deleting, updating and managing for vlan in Linux kernel. Vlanmgrd subscribes to the VLAN table and the VLAN_MEMBER table of CONFIG_DB:
- VLAN table stores the vlan-id, member port, admin_status and mtu of VLAN.
- VLAN_MEMBER table stores tagging_mod of VLAN members.

Vlanmgrd creates/deletes vlan, adds/deletes vlan member, creates and configures admin_status and mtu of vlan according to VLAN table and the VLAN_MEMBER table in CONFIG_DB, and writse the configuration to  VLAN_TABLE and VLAN_MEMBER_TABLE in APPL_DB. Portsorch subscribes to these two tables in APPL_DB and syncs the configuration to ASIC_DB, which is finally written to ASIC.
Vlanmgrd uses “ip link set” command to configure status of vlan in linux kernel. In the current implementation, admin_status of CONFIG_DB is directly used, without considering oper_status of member ports.
      The vlanmgrd structure is as follows：
      
      
![image](https://github.com/caesarkof97/SONiC/blob/jihaix/images/vlan_status_linkage_vlanmgrd_structure.png)

<p align="center"> Figure 3-1 vlanmgrd structure</p>

## 4. Problem Description

When user sets the vlan-interface status through config_db.json by editing the value of admin_status, the member_status of this vlan-interface is not considered. 
Consider the following situation, if user creates an vlan-interface with admin_status setting to “up”, but no member port is assigned, or none of its member ports is “up”, SONiC will still bring up the vlan-interface in linux kernel by executing “ip link set VlanXXX up”. This will bring some unexpected behaviors for upper APP. For example:
- Routing protocols cannot quickly detect link state change and update routes to peers.
- The upper protocol module still sends messages to the port, but the actual messages cannot be transmitted.

## 5. Target

SONiC need to consider the member_status before setting oper_status for VlanXXX in linux kernel. When member_status changed, SONiC can update oper_status automatically.

## 6. High-Level Design

### 6.1 Use oper_status to indicating the status of VlanXX in linux kernel.

Introduce oper_status to represent the status of VlanXX in linux kernel, and admin_status to represent the value in config_db.json. 
When user sets the admin_status, SONiC will check the member_status, and set oper_status according to the logic shown as follow:
```
if (member_status == “up”) && (admin_status == “up”) {
    oper_status = “up”;
} else {
    oper_status = “down”;
}
```

### 6.2 Timing when oper_status will be updated

a)	When vlan-interface is created.<br/>
b)	When member ports changed, either member add/remove, or member’s oprt_status changed.<br/>
c)	When admin_status changed, as the result of configuration changed.

### 6.3 Add CLI command to set admin_status

Add a CLI command for user to  change admin_status of vlan-interface.
```
    config vlan admin_status [vid] [up|down]
```
This command will set the admin_status stored in CONFIG_DB, and trigger an update of oper_status. Please be advised, this command will not change the value stored in config_db.json.

## 7. Detailed Design

### 7.1 New entry is added to VLAN_TABLE in APPL_DB

- member_status
- oper_status

### 7.2 UML Sequence 

![image](https://github.com/caesarkof97/SONiC/blob/jihaix/images/vlan_status_linkage_CLI_UML.jpg)

<p align="center">Figure 5-1 CLI command UML </p>



![image](https://github.com/caesarkof97/SONiC/blob/jihaix/images/vlan_status_linkage_port_state_change_UML.jpg)

<p align="center">Figure 5-2 port state change UML</p>

## 8. Unit testing

### 8.1 Test steps

1.  The oper_status of Ethernet32 and Ethernet36 are in the "up" state, the oper_status of Ethernet28 and Ethernet40 are in the "down" state, configure the VLAN and VLAN_MEMBER of |config_db.json and run the config reload command. View Vlan100 and Vlan200 and their respective member ports oper_status. The configuration is as follows:
```
"VLAN": {
        "Vlan100": {
            "vlanid": "100",
            "mtu": "9100",
            “admin_status”: “up”,
            "members":[
                "Ethernet28",
                "Ethernet32"
            ]
        },
        "Vlan200": {
            "vlanid": "200",
            "mtu": "9100",
            "admin_status": "down",
            "members":[
                "Ethernet36",
                "Ethernet40"
            ]

        }
}
"VLAN_MEMBER":{
        "Vlan100|Ethernet28":{
            "tagging_mod": "untagged"
        },
        "Vlan100|Ethernet32":{
            "tagging_mod": "untagged"
        },
        "Vlan200|Ethernet36":{
            "tagging_mod": "untagged"
        },
        "Vlan200|Ethernet40":{
            "tagging_mod": "untagged"
        }
    }
```
2.	Run “config vlan member del 100 Ethernet32”, and then run “show vlan config” to check whether the member Ethernet32 of Vlan100 has been deleted, run “ip link show Vlan100” to check oper_status of Vlan100<br/>
3.	Run “config vlan member add 100 Ethernet32”, and then run “show vlan config” to check whether the member Ethernet32 of Vlan100 has been added, run “ip link show Vlan100” to check oper_status of Vlan100<br/>
4.	Run command “ip link set Ethernet32 down”, and then run “ip link show Vlan100” to check oper_status of Vlan100<br/>
5.	Run command “ip link set Ethernet32 up”, and then run “ip link show Vlan100” to check oper_status of Vlan100<br/>
6.	Run command “ip link set Ethernet28 down”, and then run “ip link show Vlan100” to check oper_status of Vlan100<br/>
7.	Run command “ip link set Ethernet28 up”, and then run “ip link show Vlan100” to check oper_status of Vlan100<br/>
8.	Run command “config vlan admin-status 100 down”, and check admin_status of VLAN|Vlan100 of CONFIG_DB, then run “ip link show Vlan100” to check oper_status of Vlan100<br/>
9.	Run command “config vlan admin-status 100 up”, admin_status of VLAN|Vlan100 of CONFIG_DB, then run “ip link show Vlan100” to check oper_status of Vlan100

### 8.2 Expected results

1.	Result of step 1：oper_status of Vlan100, Ethernet32 and Ethernet36 are up, oper_status of Vlan200, Ethernet28 and Ethernet40 are down<br/>
2.	Result of step 2：Vlan100 member Ethernet32 was delete successfully, oper_status of Vlan100 is down<br/>
3.	Result of step 3：Vlan100 member Ethernet32 was delete successfully, oper_status of Vlan100 is up<br/>
4.	Result of step 4：oper_status of Vlan100 is down<br/>
5.	Result of step 5：oper_status of Vlan100 is up<br/>
6.	Result of step 6：oper_status of Vlan100 is up<br/>
7.	Result of step 7：oper_status of Vlan100 is up<br/>
8.	Result of step 8：admin_status of VLAN|Vlan100 of CONFIG_DB is down, oper_status of Vlan100 is down<br/>
9.	Result of step 9：admin_status of VLAN|Vlan100 of CONFIG_DB is up, oper_status of Vlan100 is up

### 8.3 Actual results

1.	Result of step 1：Pass<br/>
2.	Result of step 2：Pass<br/>
3.	Result of step 3：Pass<br/>
4.	Result of step 4：Pass<br/>
5.	Result of step 5：Pass<br/>
6.	Result of step 6：Pass<br/>
7.	Result of step 7：Pass<br/>
8.	Result of step 8：Pass<br/>
9.	Result of step 9：Pass
