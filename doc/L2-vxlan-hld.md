# L2 VXLAN HLD for SONiC

<!-- TOC -->

- [L2 VXLAN HLD for SONiC](#l2-vxlan-hld-for-sonic)
  - [1.1. Document History](#11-document-history)
  - [1.2. Abbreviations](#12-abbreviations)
  - [1.3. Brief of SONIC vxlan tunnel](#13-brief-of-sonic-vxlan-tunnel)
  - [1.4. Design considerations](#14-design-considerations)
  - [1.5. SONiC system diagram for VXLAN](#15-sonic-system-diagram-for-vxlan)
  - [1.6. Design changes](#16-design-changes)
    - [1.6.1. vxlanorch changes](#161-vxlanorch-changes)
    - [1.6.2. fdborch changes](#162-fdborch-changes)

<!-- /TOC -->

## 1.1. Document History

Version | Date | Author | Description
---------|----------|---------|-----
v.01 | 03/07/2019 | Jianjun dong | Initial version

## 1.2. Abbreviations

Term | Definition
---------|----------
VXLAN | Virtual Extensible LAN

## 1.3. Brief of SONIC vxlan tunnel

SONIC vxlan tunnel module is contributed by Microsoft.

![Figure 1.3](https://github.com/shine4chen/SONiC/blob/l2-vxlan/images/l2_vxlan_hld/L2-vxlan-HLD-figure-1.3.png)

VxlanOrch is the major subsystem for Vxlan that handles configuration request. Vxlanorch creates the tunnel and attaches encap and decap mappers. Separate tunnels are created for L2 Vxlan and L3 Vxlan and can attach different VLAN/VNI or VRF/VNI to respective tunnel.

Vxlan tunnel is created by static configuration currently. Vxlanmgrd VxlanOrch read the configuration from Config_DB, including VxlanTunnelTable and VxlanTunnelMapTable, then set these two tables in APP_DB and create L3 Vxlan tunnel in linux kernel. The formats of these two tables are list below.

```json

VXLAN_TUNNEL|{{tunnel_name}}
    "src_ip": {{ip_address}}
    "dst_ip": {{ip_address}} (OPTIONAL)

VXLAN_TUNNEL_MAP|{{tunnel_name}}|{{tunnel_map}}
    "vni": {{ vni_id}}
    "vlan": {{ vlan_id }}

```

In layer 3 Vxlan, the tunnel map is VRF to VNI. VxlanOrch and VnetOrch, are used to support Layer 3 Vxlan completely. VNET(Virtual NETwork) is beyond this document.

In layer 2 Vxlan, the tunnel map is VLAN to VNI. Layer 2 Vxlan is not supported completely, current state is TBD(To Be Done). The mainly shortcomings of L2 Vxlan are:

- Not specify the encap type when creating tunnel; The encap type is MAP_TO_INVALID, must be VLAN_ID_TO_VNI.
- Not create the bridge port for the Vxlan tunnel. FDB MAC entry can not be learnt from and set to VXLAN tunnel port.
- Not create the L2 Vxlan tunnel interface in linux kernel.

## 1.4. Design considerations

- When creating VXLAN tunnel, creating its bridge port at the same time. FDB entry can be learnt from and set to VXLAN tunnel port
- When creating VXLAN tunnel in orchagent, notify Linux kernel to create L2 VXLAN tunnel interface at the same time(L2 Vxlan tunnel interface name is different to the L3 Vxlan tunnel)

## 1.5. SONiC system diagram for VXLAN

The blocks below are just illustrating the concept flow.

![Figure 1.5](https://github.com/shine4chen/SONiC/blob/l2-vxlan/images/l2_vxlan_hld/L2-vxlan-HLD-figure-1.5.png)

**Notice: The flows represented by red arrow are newly added by this project.**

- Flow 1：vxlanmgrd reads VXLAN related configuration from CONFIG_DB, creates VXLAN tunnel and its bridge portsave parameters.
- Flow 2：vxlanmgrd send command to linux kernel to create L3 VXLAN tunnel interface, interface name is vxlan+vni, such as vxlan1000..
- Flow 3：vxlanmgrd set VXLAN tunnel configuration to APP_DB.
- Flow 4：vxlanorch reads VXLAN related configuration from APP_DB, creates VXLAN tunnel and its bridge port.
- Flow 5：vxlanorch send command to linux kernel to create L2 VXLAN tunnel interface, interface name is VTTNL****, such as VTTNL0001.
- Flow 6：APP, such as MCLAG, learns events such as L2 VXLAN tunnel interface creation, destroy, admin/link state from linux kernel.
- Flow 7：APP, such as MCLAG, exchange information with APP_DB，for example, start or stop L2 learning of MAC on VXLAN tunnel,  set FDB entry on VXLAN tunnel, etc .
- Flow 8：After configure/update APP_DB, OrchAgent monitory and process the related information, then update ASIC_DB.

## 1.6. Design changes

### 1.6.1. vxlanorch changes

Adding the following logic:

- When creating vxlan tunnel, create its bridge port. If VXLAN tunnel without bridge port, FDB MAC can not be learnt from or set to VXLAN tunnel in ASIC.
- Add the tunnel name map to counter table, so that the ‘show mac’ command can display the FDB learnt from VXLAN tunnel.
- Send command to Linux kernel to create L2 VXLAN tunnel interface.

### 1.6.2. fdborch changes

Adding the following logics:

- FDB MAC can be learnt from or set to VXLAN tunnel in ASIC.
