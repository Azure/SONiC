# MCLAG feature test plan <!-- omit in toc -->

- [overview](#overview)
  - [Scope](#Scope)
  - [Testbed](#Testbed)
  - [Topology](#Topology)
    - [Physical topology](#Physical-topology)
    - [Logical topologies](#Logical-topologies)
    - [mclag logical topology](#mclag-logical-topology)
  - [Application scenarios](#Application-scenarios)
    - [L2 scenario](#L2-scenario)
    - [L3 scenario](#L3-scenario)
    - [VXLAN scenario](#VXLAN-scenario)
- [Setup configuration](#Setup-configuration)
  - [Setup of DUT switch](#Setup-of-DUT-switch)
  - [Ansible scripts to setup and run test](#Ansible-scripts-to-setup-and-run-test)
    - [Command to deploy the topo t0-mclag](#Command-to-deploy-the-topo-t0-mclag)
    - [Deploy the initial configuration for both devices separately (with an extra parameter dut_no)](#Deploy-the-initial-configuration-for-both-devices-separately-with-an-extra-parameter-dutno)
    - [test_mclag.py and test_mclag_l3.py](#testmclagpy-and-testmclagl3py)
- [PTF Test](#PTF-Test)
  - [Input files for PTF test](#Input-files-for-ptf-test)
  - [Traffic validation in PTF](#Traffic-validation-in-ptf)
- [Test cases](#Test-cases)
  - [Summary](#Summary)
  - [L2 scenario test cases](#L2-scenario-test-cases)
    - [Test case L2#1 - mclag info check and verify data forwarding is correct through mclag](#Test-case-l21---mclag-info-check-and-verify-data-forwarding-is-correct-through-mclag)
    - [Test case L2#2 - Verify data forwarding is correct when mclag enabled interface status change](#Test-case-l22---verify-data-forwarding-is-correct-when-mclag-enabled-interface-status-change)
    - [Test case L2#3 - Verify data forwarding is correct when peer-link status change](#Test-case-l23---verify-data-forwarding-is-correct-when-peer-link-status-change)
    - [Test case L2#4 - Verify data forwarding is correct when keepalive link status change](#Test-case-l24---verify-data-forwarding-is-correct-when-keepalive-link-status-change)
    - [Test case L2#5 - Verify data forwarding is correct when peer-link and keepalive link status both change](#Test-case-l25---verify-data-forwarding-is-correct-when-peer-link-and-keepalive-link-status-both-change)
    - [Test case L2#6 - Verify data forwarding is correct when active device of mclag status change](#Test-case-l26---verify-data-forwarding-is-correct-when-active-device-of-mclag-status-change)
    - [Test case L2#7 - Verify data forwarding is correct when standby device of mclag status change](#Test-case-l27---verify-data-forwarding-is-correct-when-standby-device-of-mclag-status-change)
    - [Test case L2#8 - Verify data forwarding is correct when active devices of mclag warm-reboot](#Test-case-l28---verify-data-forwarding-is-correct-when-active-devices-of-mclag-warm-reboot)
    - [Test case L2#9 - Verify data forwarding is correct when standby devices of mclag warm-reboot](#Test-case-l29---verify-data-forwarding-is-correct-when-standby-devices-of-mclag-warm-reboot)
    - [Test case L2#10 - Verify data forwarding is correct after MAC movement](#Test-case-l210---verify-data-forwarding-is-correct-after-mac-movement)
    - [Test case L2#11 - MAC sync-up and MAC aging test](#Test-case-l211---mac-sync-up-and-mac-aging-test)
    - [Test case L2#12 - ICCP state machine test](#Test-case-l212---iccp-state-machine-test)
    - [Test case L2#13 - Scaling test](#Test-case-l213---scaling-test)
    - [Test case L2#14 - Corner test](#Test-case-l214---corner-test)
  - [L3 scenario test cases](#L3-scenario-test-cases)
    - [Test case L3#1 - mclag info check and verify data forwarding is correct through mclag](#Test-case-l31---mclag-info-check-and-verify-data-forwarding-is-correct-through-mclag)
    - [Test case L3#2 - Verify data forwarding is correct when mclag enabled interface status change](#Test-case-l32---verify-data-forwarding-is-correct-when-mclag-enabled-interface-status-change)
    - [Test case L3#3 - Verify data forwarding is correct when keepalive link status change](#Test-case-l33---verify-data-forwarding-is-correct-when-keepalive-link-status-change)
    - [Test case L3#4 - Verify data forwarding is correct when active device of mclag status change](#Test-case-l34---verify-data-forwarding-is-correct-when-active-device-of-mclag-status-change)
    - [Test case L3#5 - Verify data forwarding is correct when standby device of mclag status change](#Test-case-l35---verify-data-forwarding-is-correct-when-standby-device-of-mclag-status-change)
    - [Test case L3#6 - Verify data forwarding is correct when active devices of mclag warm-reboot](#Test-case-l36---verify-data-forwarding-is-correct-when-active-devices-of-mclag-warm-reboot)
    - [Test case L3#7 - Verify data forwarding is correct when standby devices of mclag warm-reboot](#Test-case-l37---verify-data-forwarding-is-correct-when-standby-devices-of-mclag-warm-reboot)
    - [Test case L3#8 - Scaling test](#Test-case-l38---scaling-test)
  - [VXLAN scenario test cases(vxlan test cases are for both l2 and l3)](#vxlan-scenario-test-casesvxlan-test-cases-are-for-both-l2-and-l3)
    - [Test case VXLAN#1 - mclag info check and verify data forwarding is correct through mclag](#Test-case-VXLAN1---mclag-info-check-and-verify-data-forwarding-is-correct-through-mclag)
    - [Test case VXLAN#2 - Verify data forwarding is correct when mclag enabled interface status change](#Test-case-VXLAN2---verify-data-forwarding-is-correct-when-mclag-enabled-interface-status-change)
    - [Test case VXLAN#3 - Verify data forwarding is correct when peer-link status change](#Test-case-VXLAN3---verify-data-forwarding-is-correct-when-peer-link-status-change)
    - [Test case VXLAN#4 - Verify data forwarding is correct when keepalive link status change](#Test-case-VXLAN4---verify-data-forwarding-is-correct-when-keepalive-link-status-change)
    - [Test case VXLAN#5 - Verify data forwarding is correct when peer-link and keepalive link status both change](#Test-case-VXLAN5---verify-data-forwarding-is-correct-when-peer-link-and-keepalive-link-status-both-change)
    - [Test case VXLAN#6 - Verify data forwarding is correct when active device of mclag status change](#Test-case-VXLAN6---verify-data-forwarding-is-correct-when-active-device-of-mclag-status-change)
    - [Test case VXLAN#7 - Verify data forwarding is correct when standby device of mclag status change](#Test-case-VXLAN7---verify-data-forwarding-is-correct-when-standby-device-of-mclag-status-change)
    - [Test case VXLAN#8 - Verify data forwarding is correct when active devices of mclag warm-reboot](#Test-case-VXLAN8---verify-data-forwarding-is-correct-when-active-devices-of-mclag-warm-reboot)
    - [Test case VXLAN#9 - Verify data forwarding is correct when standby devices of mclag warm-reboot](#Test-case-VXLAN9---verify-data-forwarding-is-correct-when-standby-devices-of-mclag-warm-reboot)
    - [Test case VXLAN#10 - Verify data forwarding is correct after mac movement](#Test-case-VXLAN10---verify-data-forwarding-is-correct-after-mac-movement)
    - [Test case VXLAN#11 - MAC sync-up and MAC aging test](#Test-case-VXLAN11---mac-sync-up-and-mac-aging-test)
    - [Test case VXLAN#12 - Scaling test](#Test-case-VXLAN12---scaling-test)
- [TODO](#todo)

## overview

The purpose is to test functionality of MCLAG on the SONIC switch DUT, closely resembling production environment.

### Scope

The test is running on real SONIC switch with testbed's basic configuration. The purpose of the test isn't targeting to specific class or API which are coverd by vs test cases, but functional test of MCLAG on SONIC system.

### Testbed

The test will run on the following testbed (**docker ptf must support teamd**):

- t0-mclag

### Topology

#### Physical topology

MCLAG test must use two DUTs, you can connect those two DUTs to two different leaf fanout switches, or connect those two DUTs to the same leaf fanout switch. The part in the red box needs to be added on the original SONiC test topology.

SONiC testbed overview can be found at [here](https://github.com/Azure/sonic-mgmt/blob/master/ansible/doc/README.testbed.Overview.md)

![testbed server Physical topo](https://github.com/shine4chen/SONiC/blob/mclag-test-case/images/mclag_hld/testbed-server-physical-topo.png)

#### Logical topologies

Testbeds t0-mclag is modified based on the Testbeds t0.

![testbed t0 topo](https://github.com/shine4chen/SONiC/blob/mclag-test-case/images/mclag_hld/testbed-t0-topo.png)

![testbed t0 mclag topo](https://github.com/shine4chen/SONiC/blob/mclag-test-case/images/mclag_hld/testbed-t0-mclag-topo.png)

#### mclag logical topology

![mclag logical topo](https://github.com/shine4chen/SONiC/blob/mclag-test-case/images/mclag_hld/mclag-logical-topo.png)

### Application scenarios

#### L2 scenario

All links to server ports are L2 mode, and peer-link must allow all vlans to be used. DUT is the gateway of the servers. DUT binds MCLAG's vlan interface IP address to be the same except peer-link. Establish BGP neighbors between the VMs and two DUTs. MCLAG keepalive link will be established through a separate link.

#### L3 scenario

All links to server ports are L3 mode, the peer-link is optional. DUT is the gateway of the servers. DUT binds MCLAG's interface ip address must be the same except peer-link. Establish BGP neighbors between the VMs and two DUTs. MCLAG keepalive link will be established through a separate link.

#### VXLAN scenario

All links to server ports are L2 mode. DUT is the gateway of the servers. DUT binds MCLAG's vlan interface IP address to be the same except peer-link. Establish BGP neighbors between the VMs and two DUTs. MCLAG keepalive link will be established through a separate link.

## Setup configuration

### Setup of DUT switch

Setup of SONIC DUT will be done by Ansible script. During setup Ansible will copy JSON file containing configuration for mclag to the DUT. Config utility will be used to push configuration to the SONiC DB.

#### Template <!-- omit in toc -->

```jason
"MC_LAG": {
    "<mclag-id>": {
        "local_ip": "<IP>",
        "peer_ip": "<IP>",
        "peer_link": "peer_link_name",
        "mclag_interface": "PortChanne_id"
    }
}
```

#### Attributes <!-- omit in toc -->

```jason
"<mclag-id>"      =   Description: mclag domain id
                     Attr status: mandatory
                     Value range: 1-65535
"local_ip"        =   Description: local ip address for mclag
                     Attr status: mandatory
                     Value range: A valid IPv4 address
"peer_ip"         =   Description: peer ip address for mclag
                     Attr status: mandatory
                     Value range: A valid IPv4 address
"peer_link"       =   Description: peer link Ethernet name or PortChannel name
                     Attr status: mandatory(For L3 scenario, this is optional)
                     Value range: Ethernet, PortChannel or Tunnel name
"mclag_interface" =   Description: PortChannel name which bind mclag
                     Attr status: mandatory
                     Value range: PortChannel name
```

#### Example <!-- omit in toc -->

```jason
"MC_LAG": {
    "100": {
        "local_ip": "10.100.1.1",
        "peer_ip": "10.100.1.2",
        "peer_link": "PortChannel0100",
        "mclag_interface": "PortChannel0001, PortChannel0002, PortChannel0003, PortChannel0004, PortChannel0005, PortChannel0006, PortChannel0007, PortChannel0008, PortChannel0009, PortChannel0010, PortChannel0011, PortChannel0012, PortChannel0013, PortChannel0014, PortChannel0015, PortChannel0016, PortChannel0017, PortChannel0018, PortChannel0019, PortChannel0020, PortChannel0021, PortChannel0022, PortChannel0023, PortChannel0024"
    }
}
```

### Ansible scripts to setup and run test

#### Command to deploy the topo t0-mclag

- ./testbed-cli.sh add-topo vms-t0-mclag ~/.password

#### Deploy the initial configuration for both devices separately (with an extra parameter dut_no)

- ansible-playbook -i lab config_sonic_basedon_testbed.yml -l "dut1_name" -e vm_base="VM0300" -e topo=t0-mclag -e deploy=True -e save=True -e testbed_name=vms-t0-mclag -e dut_no=1
- ansible-playbook -i lab config_sonic_basedon_testbed.yml -l "dut2_name" -e vm_base="VM0300" -e topo=t0-mclag -e deploy=True -e save=True -e testbed_name=vms-t0-mclag-dut2 -e dut_no=2

#### test_mclag.py and test_mclag_l3.py

test_mclag.py test mclag l2 scenario, and test_mclag_l3.py test mclag l3 scenario.

- pytest -vvv --disable_loganalyzer --inventory veos --host-pattern all --user admin --testbed vms-t0-mclag-16 --testbed_file testbed.csv  --duration=0 --show-capture=stdout  test_mclag.py

- pytest -vvv --disable_loganalyzer --inventory veos --host-pattern all --user admin --testbed vms-t0-mclag-16 --testbed_file testbed.csv  --duration=0 --show-capture=stdout  test_mclag_l3.py

## PTF Test

### Input files for PTF test

PTF test will generate traffic between ports and make sure it passes according to the mclag configuration. Depending on the testbed topology and the existing configuration (e.g. ECMP, LAGS, etc) packets may arrive to different ports. Therefore ports connection information will be generated from the minigraph and supplied to the PTF script. The script only cares about whether all packets are received on the correct port, not the rate at which they are sent(Our current server runs scripts at hundreds of packets per second).

### Traffic validation in PTF

Depending on the test PTF test will verify the packet arrived or dropped.

## Test cases

### Summary

| NO. | Test case info |

| -: | - | - |

[L2 scenario test cases](#L2-scenario-test-cases) |

| 1 | [mclag info check and verify data forwarding is correct through mclag](#Test-case-L21---mclag-info-check-and-verify-data-forwarding-is-correct-through-mclag) |

| 2 | [Verify data forwarding is correct when mclag enabled interface status change](#Test-case-L22---Verify-data-forwarding-is-correct-when-mclag-enabled-interface-status-change) |

| 3 | [Verify data forwarding is correct when peer-link status change](#Test-case-L23---Verify-data-forwarding-is-correct-when-peer-link-status-change) |

| 4 | [Verify data forwarding is correct when keepalive link status change](#Test-case-L24---Verify-data-forwarding-is-correct-when-Keepalive-link-status-change) |

| 5 | [Verify data forwarding is correct when peer-link and keepalive link status both change](#Test-case-L25---Verify-data-forwarding-is-correct-when-peer-link-and-Keepalive-link-status-both-change) |

| 6 | [Verify data forwarding is correct when active device of mclag status change](#Test-case-L26---Verify-data-forwarding-is-correct-when-active-device-of-mclag-status-change) |

| 7 | [Verify data forwarding is correct when standby device of mclag status change](#Test-case-L27---Verify-data-forwarding-is-correct-when-standby-device-of-mclag-status-change) |

| 8 | [Verify data forwarding is correct when active devices of mclag warm-reboot](#Test-case-L28---Verify-data-forwarding-is-correct-when-active-devices-of-mclag-warm-reboot) |

| 9 | [Verify data forwarding is correct when standby devices of mclag warm-reboot](#Test-case-L29---Verify-data-forwarding-is-correct-when-standby-devices-of-mclag-warm-reboot) |

| 10 | [Verify data forwarding is correct after MAC movement](#Test-case-L210---Verify-data-forwarding-is-correct-after-MAC-movement) |

| 11 | [MAC sync-up and MAC aging test](#Test-case-L211---MAC-sync-up-and-MAC-agin-test) |

| 12 | [ICCP state machine test](#Test-case-L212---ICCP-state-machine-test) |

| 13 | [Scaling test](#Test-case-L213---Scaling-test) |

| 14 | [Corner test](#Test-case-L214---Corner-test) |

[L3 scenario test cases](#L3-scenario-test-cases) |

| 1 | [mclag info check and verify data forwarding is correct through mclag](#Test-case-L31---mclag-info-check-and-verify-data-forwarding-is-correct-through-mclag) |

| 2 | [Verify data forwarding is correct when mclag enabled interface status change](#Test-case-L32---Verify-data-forwarding-is-correct-when-mclag-enabled-interface-status-change) |

| 3 | [Verify data forwarding is correct when keepalive link status change](#Test-case-L33---Verify-data-forwarding-is-correct-when-keepalive-link-status-change) |

| 4 | [Verify data forwarding is correct when active device of mclag status change](#Test-case-L34---Verify-data-forwarding-is-correct-when-active-device-of-mclag-status-change) |

| 5 | [Verify data forwarding is correct when standby device of mclag status change](#Test-case-L35---Verify-data-forwarding-is-correct-when-standby-device-of-mclag-status-change) |

| 6 | [Verify data forwarding is correct when active devices of mclag warm-reboot](#Test-case-L36---Verify-data-forwarding-is-correct-when-active-devices-of-mclag-warm-reboot) |

| 7 | [Verify data forwarding is correct when standby devices of mclag warm-reboot](#Test-case-L37---Verify-data-forwarding-is-correct-when-standby-devices-of-mclag-warm-reboot) |

| 8 | [Scaling test](#Test-case-L38---Scaling-test) |

[VXLAN scenario test cases](#VXLAN-scenario-test-cases) |

| 1 | [mclag info check and verify data forwarding is correct through mclag](#Test-case-VXLAN1---mclag-info-check-and-verify-data-forwarding-is-correct-through-mclag) |

| 2 | [Verify data forwarding is correct when mclag enabled interface status change](#Test-case-VXLAN2---Verify-data-forwarding-is-correct-when-mclag-enabled-interface-status-change) |

| 3 | [Verify data forwarding is correct when peer-link status change](#Test-case-VXLAN3---Verify-data-forwarding-is-correct-when-peer-link-status-change) |

| 4 | [Verify data forwarding is correct when keepalive link status change](#Test-case-VXLAN4---Verify-data-forwarding-is-correct-when-Keepalive-link-status-change) |

| 5 | [Verify data forwarding is correct when peer-link and keepalive link status both change](#Test-case-VXLAN5---Verify-data-forwarding-is-correct-when-peer-link-and-Keepalive-link-status-both-change) |

| 6 | [Verify data forwarding is correct when active device of mclag status change](#Test-case-VXLAN6---Verify-data-forwarding-is-correct-when-active-device-of-mclag-status-change) |

| 7 | [Verify data forwarding is correct when standby device of mclag status change](#Test-case-VXLAN7---Verify-data-forwarding-is-correct-when-standby-device-of-mclag-status-change) |

| 8 | [Verify data forwarding is correct when active devices of mclag warm-reboot](#Test-case-VXLAN8---Verify-data-forwarding-is-correct-when-active-devices-of-mclag-warm-reboot) |

| 9 | [Verify data forwarding is correct when standby devices of mclag warm-reboot](#Test-case-VXLAN9---Verify-data-forwarding-is-correct-when-standby-devices-of-mclag-warm-reboot) |

| 10 | [Verify data forwarding is correct after MAC movement](#Test-case-VXLAN10---Verify-data-forwarding-is-correct-after-MAC-movement) |

| 11 | [MAC sync-up and MAC aging test](#Test-case-VXLAN11---MAC-sync-up-and-MAC-aging-test) |

| 12 | [Scaling test](#Test-case-VXLAN12---Scaling-test) |

### L2 scenario test cases

#### SetUp: Load mclag L2 configuration files to dut and load it  <!-- omit in toc -->

#### Test case L2#1 - mclag info check and verify data forwarding is correct through mclag

##### Test objective <!-- omit in toc -->

Verify that mclag info is correct when mclag build ok.

Verify data forwarding is correct through mclag when mclag build ok.

##### Test steps <!-- omit in toc -->

1. Verify ping succeed between mclag peers.
2. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive is ok.
3. Verify standby device changes its LACP system ID to be the same as active device.
4. Servers(including orphan ports) send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
5. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
6. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L2#2 - Verify data forwarding is correct when mclag enabled interface status change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when mclag enabled interface status change.

##### Test steps <!-- omit in toc -->

1. mclag enabled interface status change to down(shutdown port).
2. Servers(exclude down ports) send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
3. "show mac" command verify the macs learned on down port before will point to the peer-link.
4. "mclagdctl -i \<mclag-id\> dump mac" command verify the macs learned on down port before will add "L" flag and add "P" flag indicates the same MAC entry in peer devic.
5. "show arp" command verify the arps learned on down port before will point to the peer-link.
6. mclag enabled interface status recover to up(startup port).
7. Servers send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
8. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
9. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L2#3 - Verify data forwarding is correct when peer-link status change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when peer-link status change.

##### Test steps <!-- omit in toc -->

1. Peer-link status change to down(shutdown peer-link).
2. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive is still ok.
3. Servers(exclude orphan ports) send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports. Servers(orphan port) can only communicate with other orphan port servers connected to the same device, we will not verify the data forwarding on orphan port servers in this case.
4. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device.
5. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device.
6. Peer-link status recover to up(startup peer-link).
7. Servers send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
8. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
9. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L2#4 - Verify data forwarding is correct when keepalive link status change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when keepalive link status change.

##### Test steps <!-- omit in toc -->

1. Keepalive link status change to down(shutdown keepalive link).
2. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive change to error.
3. Verify standby device changes its LACP system ID to the local default
4. Servers(exclude ports which link to standby device of mclag) send packets to others Servers(Send L2 packets to servers that belong to the same vlan and L3 packets to servers that belong to other vlans). All packets must be received on the correct destination ports.
5. "show mac" command verify the macs learned on standby device orphan ports will be deleted.
6. "show arp" command verify the arps learned on standby device orphan ports will be deleted.
7. Keepalive link status recover to up(startup keepalive link).
8. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive change to ok.
9. Servers send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
10. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
11. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L2#5 - Verify data forwarding is correct when peer-link and keepalive link status both change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when peer-link and keepalive link status both change.

##### Test steps <!-- omit in toc -->

1. Peer-link and keepalive link status change to down(shutdown the peer-link and keepalive link).
2. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive change to error.
3. Verify standby device changes its LACP system ID to the local default
4. Servers(exclude orphan ports and those ports link to standby device of mclag) send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
5. "show mac" command verify the macs learned on standby device orphan ports will be deleted.
6. "show arp" command verify the arps learned on standby device orphan ports will be deleted.
7. Peer-link and keepalive link status recover to up(startup the peer-link and keepalive link).
8. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive change to ok.
9. Servers send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
10. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
11. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L2#6 - Verify data forwarding is correct when active device of mclag status change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when active device of mclag status change.

##### Test steps <!-- omit in toc -->

1. Active device of mclag status change to down(reboot active device of mclag).
2. "mclagdctl -i \<mclag-id\> dump state" command on standby device verify mclag keepalive change to error.
3. Servers(exclude ports which link to active device of mclag) send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
4. "show mac" command on standby device verify the macs learned on active device orphan ports will be deleted.
5. "show arp" command on standby device verify the arps learned on active device orphan ports will be deleted.
6. Active device of mclag status recover to up.
7. "mclagdctl -i \<mclag-id\> dump state" command on standby device verify mclag keepalive change to ok.
8. Servers send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
9. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
10. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L2#7 - Verify data forwarding is correct when standby device of mclag status change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when standby device of mclag status change.

##### Test steps <!-- omit in toc -->

1. Standby device of mclag status change to down(reboot standby device of mclag).
2. "mclagdctl -i \<mclag-id\> dump state" command on active device verify mclag keepalive change to error.
3. Servers(exclude ports which link to standby device of mclag) send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
4. "show mac" command on active device verify the macs learned on standby device orphan ports will be deleted.
5. "show arp" command on active device verify the arps learned on standby device orphan ports will be deleted.
6. Standby device of mclag status recover to up.
7. "mclagdctl -i \<mclag-id\> dump state" command on active device verify mclag keepalive change to ok.
8. Servers send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
9. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
10. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L2#8 - Verify data forwarding is correct when active devices of mclag warm-reboot

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when active devices of mclag warm-reboot.

##### Test steps <!-- omit in toc -->

1. Active device of mclag warm-reboot.
2. Servers send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
3. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
4. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L2#9 - Verify data forwarding is correct when standby devices of mclag warm-reboot

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when standby devices of mclag warm-reboot.

##### Test steps <!-- omit in toc -->

1. Standby device of mclag warm-reboot.
2. Servers send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
3. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
4. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L2#10 - Verify data forwarding is correct after MAC movement

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct after MAC movement.

##### Test steps <!-- omit in toc -->

1. Servers(including orphan ports) send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
2. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
3. "mclagdctl -i \<mclag-id\> dump mac" command verify the macs learned on local device orphan ports will add "P" flag and add "L" flag indicates the same MAC entry in peer devic. The macs learned on mclag enabled interfaces doesn't have any flags.
4. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.
5. Modify ptf configruation move one orphan port server(server1) from active device to standby device.
6. Verify server1 ping dut succeed.
7. "show mac" command verify the server1 mac point to the peer-link on mclag active device. The server1 mac learned on orphan ports of mclag standby device.
8. "mclagdctl -i \<mclag-id\> dump mac" command verify "L" flag indicates the server1 MAC on active device and "P" flag indicates the server1 MAC on standby device.
9. Servers(including orphan ports) send packets to server1(servers belonging to the same vlan send L2 packets, servers belonging to other vlans send L3 packets). All packets must be received on the correct destination ports.
10. Recover ptf configruation to default.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L2#11 - MAC sync-up and MAC aging test

##### Test objective <!-- omit in toc -->

Verify MAC sync-up and MAC aging ok.

##### Test steps <!-- omit in toc -->

1. mclag enabled interface status change to down(shutdown port).
2. "mclagdctl -i \<mclag-id\> dump mac" command verify the macs learned on down port before will add "L" flag and add "P" flag indicates the same MAC entry in peer devic.
3. mclag enabled interface status change to up(startup port).
4. "mclagdctl -i \<mclag-id\> dump mac" command verify those macs do not has any flag.
5. Servers send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
6. "mclagdctl -i \<mclag-id\> dump mac" command verify the macs learned on local device orphan ports will add "P" flag and add "L" flag indicates the same MAC entry in peer devic. The macs learned on mclag enabled interfaces doesn't have any flags.
7. Modify mac aging time to 60s.
8. Servers(including orphan ports) send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
9. "mclagdctl -i \<mclag-id\> dump mac" command verify the macs learned on local device orphan ports will add "P" flag and add "L" flag indicates the same MAC entry in peer devic. The macs learned on mclag enabled interfaces doesn't have any flags.
10. Wait 30s. Servers(exclude ports which link to standby device of mclag) send packets to others Servers(The purpose was to keep the MAC from aging on the active device).
11. Wait 30s. The macs learned on standby device will aging. Active device: "mclagdctl -i \<mclag-id\> dump mac" command verify the macs learned on mclag enable interface will add "P" flag, add the macs learned on orphan ports of standby device will be deleted. Standby device: "mclagdctl -i \<mclag-id\> dump mac" command verify the macs learned on mclag enable interface will add "L" flag, add the macs learned on orphan ports of active device will keep "L" flag.
12. Servers(exclude ports which link to active device of mclag) send packets to others Servers.
13. Active device: "mclagdctl -i \<mclag-id\> dump mac" command verify the macs learned on mclag enable interface will keep "P" flag, add the macs learned on orphan ports of standby device will add "P" flag. Standby device: "mclagdctl -i \<mclag-id\> dump mac" command verify the macs learned on mclag enable interface will keep "L" flag, add the macs learned on orphan ports of active device will keep "L" flag.
14. Modify mac aging time to default 600s.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L2#12 - ICCP state machine test

##### Test objective <!-- omit in toc -->

Verify ICCP state machine in syslog.

##### Test steps <!-- omit in toc -->

1. Reboot the active device.
2. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive is ok on active device.
3. ICCP state machine info can be find in syslog(from NONEXISTENT to INITIALIZED; from XXX to CAPREC; from XXX to OPERATIONAL).
4. Reboot the standby device.
5. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive is ok on standby device.
6. ICCP state machine info can be find in syslog(from NONEXISTENT to INITIALIZED; from XXX to CAPREC; from XXX to OPERATIONAL).

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L2#13 - Scaling test

##### Test objective <!-- omit in toc -->

MAC and ARP Scaling test.

##### Test steps <!-- omit in toc -->

1. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive is ok.
2. Each server(including orphan ports) simulate 100 virtual machines.
3. Servers(including orphan ports) send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
4. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
5. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.
6. Recover server configruation

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L2#14 - Corner test

##### Test objective <!-- omit in toc -->

Corner test for teamd service hangs.

##### Test steps <!-- omit in toc -->

1. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive is ok.
2. Servers(including orphan ports) send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
3. "sudo service teamd stop" on active device.
4. Wait 90s. The server port connected to the active device changes to the deselected state. The mclag peer-link break.
5. Servers(exclude orphan ports and those ports link to active device of mclag) send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
6. "sudo service teamd start; sudo service swss restart" on active device.
7. The server port connected to the active device changes to the selected state. The mclag peer-link recover up.
8. Servers(including orphan ports) send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
9. "sudo service teamd stop" on standby device.
10. Wait 90s. The server port connected to the standby device changes to the deselected state. The mclag peer-link break.
11. Servers(exclude orphan ports and those ports link to standby device of mclag) send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.
12. "sudo service teamd start; sudo service swss restart" on standby device.
13. The server port connected to the standby device changes to the selected state. The mclag peer-link recover up.
14. Servers(including orphan ports) send packets to others Servers and VMs(send L2 packets to servers that belong to the same vlan, and send L3 packets to VMs and servers belonging to other vlans). All packets must be received on the correct destination ports.

##### Expected test result: Pass <!-- omit in toc -->

#### TearDown: Set default configuration <!-- omit in toc -->

### L3 scenario test cases

#### SetUp: Load mclag L3 configuration files to dut and load it <!-- omit in toc -->

#### Test case L3#1 - mclag info check and verify data forwarding is correct through mclag

##### Test objective <!-- omit in toc -->

Verify that mclag info is correct when mclag build ok.

Verify data forwarding is correct through mclag when mclag build ok.

##### Test steps <!-- omit in toc -->

1. Verify ping succeed between mclag peers.
2. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive is ok.
3. Verify standby device changes its LACP system ID to be the same as active device.
4. Servers(including orphan ports) send packets to others Servers and VMs(send L3 packets to other servers and VMs). All packets must be received on the correct destination ports.
5. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device(Services on orphan ports forwarding support by routing protocol or static route).

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L3#2 - Verify data forwarding is correct when mclag enabled interface status change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when mclag enabled interface status change.

##### Test steps <!-- omit in toc -->

1. mclag enabled interface status change to down(shutdown port).
2. Servers(exclude down ports) send packets to others Servers and VMs(send L3 packets to other servers and VMs). All packets must be received on the correct destination ports.
3. mclag enabled interface status recover to up(startup port).
4. Servers send packets to others Servers and VMs(send L3 packets to other servers and VMs). All packets must be received on the correct destination ports.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L3#3 - Verify data forwarding is correct when keepalive link status change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when keepalive link status change.

##### Test steps <!-- omit in toc -->

1. Keepalive link status change to down(shutdown keepalive link).
2. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive change to error.
3. Verify standby device changes its LACP system ID to the local default
4. Servers(exclude ports which link to standby device of mclag) send packets to others Servers(send L3 packets to other servers). All packets must be received on the correct destination ports.
5. Keepalive link status recover to up(startup keepalive link).
6. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive change to ok.
7. Servers send packets to others Servers and VMs(send L3 packets to other servers and VMs). All packets must be received on the correct destination ports.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L3#4 - Verify data forwarding is correct when active device of mclag status change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when active device of mclag status change.

##### Test steps <!-- omit in toc -->

1. Active device of mclag status change to down(reboot active device of mclag).
2. "mclagdctl -i \<mclag-id\> dump state" command on standby device verify mclag keepalive change to error.
3. Servers(exclude ports which link to active device of mclag) send packets to others Servers and VMs(send L3 packets to other servers and VMs). All packets must be received on the correct destination ports.
4. Active device of mclag status recover to up.
5. "mclagdctl -i \<mclag-id\> dump state" command on standby device verify mclag keepalive change to ok.
6. Servers send packets to others Servers and VMs(send L3 packets to other servers and VMs). All packets must be received on the correct destination ports.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L3#5 - Verify data forwarding is correct when standby device of mclag status change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when standby device of mclag status change.

##### Test steps <!-- omit in toc -->

1. Standby device of mclag status change to down(reboot standby device of mclag).
2. "mclagdctl -i \<mclag-id\> dump state" command on active device verify mclag keepalive change to error.
3. Servers(exclude ports which link to standby device of mclag) send packets to others Servers and VMs(send L3 packets to other servers and VMs). All packets must be received on the correct destination ports.
4. Standby device of mclag status recover to up.
5. "mclagdctl -i \<mclag-id\> dump state" command on active device verify mclag keepalive change to ok.
6. Servers send packets to others Servers and VMs(send L3 packets to other servers and VMs). All packets must be received on the correct destination ports.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L3#6 - Verify data forwarding is correct when active devices of mclag warm-reboot

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when active devices of mclag warm-reboot.

##### Test steps <!-- omit in toc -->

1. Active device of mclag warm-reboot.
2. Servers send packets to others Servers and VMs(send L3 packets to other servers and VMs). All packets must be received on the correct destination ports.
3. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L3#7 - Verify data forwarding is correct when standby devices of mclag warm-reboot

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when standby devices of mclag warm-reboot.

##### Test steps <!-- omit in toc -->

1. Standby device of mclag warm-reboot.
2. Servers send packets to others Servers and VMs(send L3 packets to other servers and VMs). All packets must be received on the correct destination ports.
3. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case L3#8 - Scaling test

##### Test objective <!-- omit in toc -->

ARP Scaling test.

##### Test steps <!-- omit in toc -->

1. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive is ok.
2. Each server(including orphan ports) simulate 100 virtual machines.
3. Servers(including orphan portss) send packets to others Servers and VMs(send L3 packets to other servers and VMs). All packets must be received on the correct destination ports.
4. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device.
5. Recover server configruation

##### Expected test result: Pass <!-- omit in toc -->

#### TearDown: Set default configuration <!-- omit in toc -->

### VXLAN scenario test cases(vxlan test cases are for both l2 and l3)

#### SetUp: Load mclag VXLAN configuration files to dut and load it  <!-- omit in toc -->

#### Test case VXLAN#1 - mclag info check and verify data forwarding is correct through mclag

##### Test objective <!-- omit in toc -->

Verify that mclag info is correct when mclag build ok.

Verify data forwarding is correct through mclag when mclag build ok.

##### Test steps <!-- omit in toc -->

1. Verify ping succeed between mclag peers.
2. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive is ok.
3. Verify standby device changes its LACP system ID to be the same as active device.
4. Servers(including orphan ports) send packets to others Servers(send L2 packets to servers that belong to the same VNI, and send L3 packets to VMs and servers belonging to other VNI). All packets must be received on the correct destination ports. VXLAN tunnel encapsulated packets can be captured on the link VM ports.
5. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
6. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case VXLAN#2 - Verify data forwarding is correct when mclag enabled interface status change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when mclag enabled interface status change.

##### Test steps <!-- omit in toc -->

1. mclag enabled interface status change to down(shutdown port).
2. Servers(exclude down ports) send packets to others Servers(send L2 packets to servers that belong to the same VNI, and send L3 packets to VMs and servers belonging to other VNI). All packets must be received on the correct destination ports.
3. "show mac" command verify the macs learned on down port before will point to the peer-link.
4. "show arp" command verify the arps learned on down port before will point to the peer-link.
5. mclag enabled interface status recover to up(startup port).
6. Servers send packets to others Servers. All packets must be received on the correct destination ports.
7. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
8. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case VXLAN#3 - Verify data forwarding is correct when peer-link status change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when peer-link status change.

##### Test steps <!-- omit in toc -->

1. Peer-link status change to down(shutdown peer-link, but not support now).
2. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive is still ok.
3. Servers(exclude orphan ports) send packets to others Servers(send L2 packets to servers that belong to the same VNI, and send L3 packets to VMs and servers belonging to other VNI). All packets must be received on the correct destination ports. Servers(orphan port) can only communicate with other orphan port servers connected to the same device, we will not verify the data forwarding on orphan port servers in this case.
4. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device.
5. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device.
6. Peer-link status recover to up(startup peer-link, but not support now).
7. Servers send packets to others Servers. All packets must be received on the correct destination ports.
8. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
9. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Failed(The peer-link cannot shutdown in the VXLAN scenario, This test case can pass after BFD function is introduced) <!-- omit in toc -->

#### Test case VXLAN#4 - Verify data forwarding is correct when keepalive link status change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when keepalive link status change.

##### Test steps <!-- omit in toc -->

1. Keepalive link status change to down(shutdown keepalive link).
2. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive change to error.
3. Verify standby device changes its LACP system ID to the local default
4. Servers(exclude ports which link to standby device of mclag) send packets to others Servers(send L2 packets to servers that belong to the same VNI, and send L3 packets to VMs and servers belonging to other VNI). All packets must be received on the correct destination ports.
5. "show mac" command verify the macs learned on standby device orphan ports will be deleted.
6. "show arp" command verify the arps learned on standby device orphan ports will be deleted.
7. Keepalive link status recover to up(startup keepalive link).
8. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive change to ok.
9. Servers send packets to others Servers. All packets must be received on the correct destination ports.
10. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
11. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case VXLAN#5 - Verify data forwarding is correct when peer-link and keepalive link status both change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when peer-link and keepalive link status both change.

##### Test steps <!-- omit in toc -->

1. Peer-link and Keepalive link status change to down(shutdown the peer-link and keepalive link, but not support shutdown the peer-link now).
2. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive change to error.
3. Verify standby device changes its LACP system ID to the local default
4. Servers(exclude orphan ports and those ports link to standby device of mclag) send packets to others Servers(send L2 packets to servers that belong to the same VNI, and send L3 packets to VMs and servers belonging to other VNI). All packets must be received on the correct destination ports.
5. "show mac" command verify the macs learned on standby device orphan ports will be deleted.
6. "show arp" command verify the arps learned on standby device orphan ports will be deleted.
7. Peer-link and keepalive link status recover to up(startup the peer-link and keepalive link, but not support startup the peer-link now).
8. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive change to ok.
9. Servers send packets to others Servers. All packets must be received on the correct destination ports.
10. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
11. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Failed(The peer-link cannot shutdown in the VXLAN scenario. This test case can pass after BFD function is introduced) <!-- omit in toc -->

#### Test case VXLAN#6 - Verify data forwarding is correct when active device of mclag status change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when active device of mclag status change.

##### Test steps <!-- omit in toc -->

1. Active device of mclag status change to down(reboot active device of mclag).
2. "mclagdctl -i \<mclag-id\> dump state" command on standby device verify mclag keepalive change to error.
3. Servers(exclude ports which link to active device of mclag) send packets to others Servers(send L2 packets to servers that belong to the same VNI, and send L3 packets to VMs and servers belonging to other VNI). All packets must be received on the correct destination ports.
4. "show mac" command on standby device verify the macs learned on active device orphan ports will be deleted.
5. "show arp" command on standby device verify the arps learned on active device orphan ports will be deleted.
6. Active device of mclag status recover to up.
7. "mclagdctl -i \<mclag-id\> dump state" command on standby device verify mclag keepalive change to ok.
8. Servers send packets to others Servers. All packets must be received on the correct destination ports.
9. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
10. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case VXLAN#7 - Verify data forwarding is correct when standby device of mclag status change

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when standby device of mclag status change.

##### Test steps <!-- omit in toc -->

1. Standby device of mclag status change to down(reboot standby device of mclag).
2. "mclagdctl -i \<mclag-id\> dump state" command on active device verify mclag keepalive change to error.
3. Servers(exclude ports which link to standby device of mclag) send packets to others Servers(send L2 packets to servers that belong to the same VNI, and send L3 packets to VMs and servers belonging to other VNI). All packets must be received on the correct destination ports.
4. "show mac" command on active device verify the macs learned on standby device orphan ports will be deleted.
5. "show arp" command on active device verify the arps learned on standby device orphan ports will be deleted.
6. Standby device of mclag status recover to up.
7. "mclagdctl -i \<mclag-id\> dump state" command on active device verify mclag keepalive change to ok.
8. Servers send packets to others Servers. All packets must be received on the correct destination ports.
9. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
10. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case VXLAN#8 - Verify data forwarding is correct when active devices of mclag warm-reboot

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when active devices of mclag warm-reboot.

##### Test steps <!-- omit in toc -->

1. Active device of mclag warm-reboot.
2. Servers send packets to others Servers(send L2 packets to servers that belong to the same VNI, and send L3 packets to VMs and servers belonging to other VNI). All packets must be received on the correct destination ports.
3. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
4. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case VXLAN#9 - Verify data forwarding is correct when standby devices of mclag warm-reboot

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct when standby devices of mclag warm-reboot.

##### Test steps <!-- omit in toc -->

1. Standby device of mclag warm-reboot.
2. Servers send packets to others Servers(send L2 packets to servers that belong to the same VNI, and send L3 packets to VMs and servers belonging to other VNI). All packets must be received on the correct destination ports.
3. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
4. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case VXLAN#10 - Verify data forwarding is correct after mac movement

##### Test objective <!-- omit in toc -->

Verify data forwarding is correct after mac movement.

##### Test steps <!-- omit in toc -->

1. Servers(including orphan ports) send packets to others Servers(send L2 packets to servers that belong to the same VNI, and send L3 packets to VMs and servers belonging to other VNI). All packets must be received on the correct destination ports.
2. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
3. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.
4. Modify ptf configruation move one orphan port server(server1) from active device to standy device.
5. Verify server1 ping dut succeed.
6. "show mac" command verify the server1 mac point to the peer-link on mclag active device. The server1 mac learned on orphan ports of mclag standby device.
7. Servers(including orphan ports) send packets to server1. All packets must be received on the correct destination ports.
8. Recover ptf configruation to default.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case VXLAN#11 - MAC sync-up and MAC aging test

##### Test objective <!-- omit in toc -->

Verify MAC sync-up and MAC aging ok.

##### Test steps <!-- omit in toc -->

1. Modify mac aging time to 60s.
2. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive is ok.
3. Servers(including orphan ports) send packets to others Servers and VMs(send L2 packets to servers that belong to the same VNI, and send L3 packets to VMs and servers belonging to other VNI). All packets must be received on the correct destination ports.
4. "mclagdctl -i \<mclag-id\> dump mac" command verify the macs learned on local device orphan ports will add "P" flag and add "L" flag indicates the same MAC entry in peer devic. The macs learned on mclag enabled interfaces doesn't have any flags.
5. Wait 30s. Servers(exclude ports which link to standby device of mclag) send packets to others Servers(The purpose was to keep the MAC from aging on the active device).
6. Wait 30s. The macs learned on standby device will aging. Active device: "mclagdctl -i \<mclag-id\> dump mac" command verify the macs learned on mclag enable interface will add "P" flag, add the macs learned on orphan ports of standby device will be deleted. Standby device: "mclagdctl -i \<mclag-id\> dump mac" command verify the macs learned on mclag enable interface will add "L" flag, add the macs learned on orphan ports of active device will keep "L" flag.
7. Servers(exclude ports which link to active device of mclag) send packets to others Servers.
8. "mclagdctl -i \<mclag-id\> dump mac" command verify the macs learned on local device orphan ports will add "P" flag and add "L" flag indicates the same MAC entry in peer devic. The macs learned on mclag enabled interfaces doesn't have any flags.
9. mclag enabled interface status change to down(shutdown port).
10. "mclagdctl -i \<mclag-id\> dump mac" command verify the macs learned on down port before will add "L" flag and add "P" flag indicates the same MAC entry in peer devic.
11. mclag enabled interface status change to up(startup port).
12. "mclagdctl -i \<mclag-id\> dump mac" command verify the macs not change.
13. Servers send packets to others Servers and VMs(send L2 packets to servers that belong to the same VNI, and send L3 packets to VMs and servers belonging to other VNI). All packets must be received on the correct destination ports.
14. "mclagdctl -i \<mclag-id\> dump mac" command verify the macs learned on local device orphan ports will add "P" flag and add "L" flag indicates the same MAC entry in peer devic. The macs learned on mclag enabled interfaces doesn't have any flags.
15. Modify mac aging time to default 600s.

##### Expected test result: Pass <!-- omit in toc -->

#### Test case VXLAN#12 - Scaling test

##### Test objective <!-- omit in toc -->

MAC and ARP Scaling test.

##### Test steps <!-- omit in toc -->

1. "mclagdctl -i \<mclag-id\> dump state" command verify mclag keepalive is ok.
2. Each server(including orphan ports) simulate 100 virtual machines.
3. Servers(including orphan ports) send packets to others Servers and VMs(send L2 packets to servers that belong to the same VNI, and send L3 packets to VMs and servers belonging to other VNI). All packets must be received on the correct destination ports.
4. "show mac" command verify the macs learned on mclag enabled interface are the same on both mclag device. The macs learned on orphan ports will point to the peer-link on another device.
5. "show arp" command verify the arps learned on mclag enabled interface are the same on both mclag device. The arps learned on orphan ports will point to the peer-link on another device.
6. Recover server configruation

##### Expected test result: Pass <!-- omit in toc -->

#### TearDown: Set default configuration <!-- omit in toc -->

## TODO

VXLAN scenario will Submit as phase 2.

The peer-link cannot shutdown in the VXLAN scenario. This test case can pass after the BFD function is introduced, and the following two test cases will fail at present.

- Test case VXLAN#4 - Verify data forwarding is correct when peer-link status change
- Test case VXLAN#5 - Verify data forwarding is correct when peer-link and keepalive link status both change
