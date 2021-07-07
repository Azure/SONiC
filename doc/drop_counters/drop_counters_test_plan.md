# Configurable Packet Drop Counters Test Plan

## Table of Contents
- [1 Overview](#1-overview)
    - [1.1 Scope](#11-scope)
    - [1.2 Test Cases](#12-test-cases)
    - [1.3 Related CLI Commands](#13-related-cli-commands)
- [2 Testing Parameters](#2-testing-parameters)
    - [2.1 Supported Topologies](#21-supported-topologies)
    - [2.2 Testbed Constraints](#22-testbed-constraints)
    - [2.3 IPv6 Support](#23-ipv6-support)
- [3 Testing Flow](#3-testing-flow)
    - [3.1 Setup](#31-setup)
    - [3.2 Testing](#32-testing)
    - [3.3 Teardown](#33-teardown)
- [4 Test Cases](#4-test-cases)
    - [4.1 Capabilities](#41-capabilities)
    - [4.2 Individual Pipeline Drops](#42-individual-pipeline-drops)
    - [4.3 Multiple Pipeline Drops](#43-multiple-pipeline-drops)
    - [4.4 Multiple Counters](#44-multiple-counters)
    - [4.5 Configuration](#45-configuration)
    - [4.6 Regressions](#46-regressions)
- [5 Future Improvements](#5-future-improvements)
- [6 Acknowledgements](#6-acknowledgements)
- [7 References](#7-references)

## 1 Overview
The purpose of this test is to validate that the configurable drop counters work as described in the HLD and as described by the SONiC command reference.

### 1.1 Scope
The main goal of these test cases is to validate that:
1. Drop counter capabilities can be queried
    - Relevant capability information is added to State DB
    - Relevant capability information can be accessed via the command line
    - Empty queries (e.g. from devices that do not support drop counters) do not cause syncd or orchagent to crash
    - Failed queries (e.g. from devices that do not support SAI query APIs) do not cause syncd or orchagent to crash
2. Drop counters can be installed and used as expected
    - Drop counters correctly count drops that occur due to reasons they are configured to track
    - Drop counters do not count drops that occur due to reasons they are _not_ configured to track
    - Drop counters do not count any other packets that are not dropped (e.g. control packets that are trapped to the CPU)
    - Drop counters do not double-count packets that fulfill multiple drop reasons
3. Drop counters can be re-configured (e.g. reasons can be added and removed from the drop counters)
    - Drop counters continue to correctly count drops after they have been re-configured (following the criteria described above)
    - Drop counters are cleared upon re-configuration
4. Drop counters can be deleted
    - Drop counters do not continue to count after being deleted
    - Drop counter counts are not retained if a counter is created after a counter has been deleted (e.g. new counters always start at 0)
5. Multiple drop counters can be configured without interfering with each other
    - Creating two counters with overlapping drop reasons does not result in double counting on either counter

This test plan does not cover:
1. Drops that are related to L2 packet corruption.
2. Drops that are related to congestion.
3. Edge cases in the CLI - this is deferred to the CLI tests provided in sonic-utilities.

### 1.2 Test Cases
| Test Case | Category | Short Description | Drop Reasons |
|-----------|---------------------------|--------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| 4.1.1 | Capabilities | Verify that drop counter capabilities can be queried | N/A |
| 4.2.1 | Individual Pipeline Drops | Verify that SMAC_EQUALS_DMAC is counted correctly | SMAC_EQUALS_DMAC |
| 4.2.2 | Individual Pipeline Drops | Verify that ACL_ANY is counted correctly | ACL_ANY |
| 4.2.3 | Individual Pipeline Drops | Verify that SIP_LINK_LOCAL is counted correctly | SIP_LINK_LOCAL |
| 4.2.4 | Individual Pipeline Drops | Verify that DIP_LINK_LOCAL is counted correctly | DIP_LINK_LOCAL |
| 4.2.5 | Individual Pipeline Drops | Verify that L3_EGRESS_LINK_DOWN is counted correctly | L3_EGRESS_LINK_DOWN |
| 4.2.6 | Individual Pipeline Drops | Verify that UNRESOLVED_NEXT_HOP is counted correctly | UNRESOLVED_NEXT_HOP |
| 4.3.1 | Multiple Pipeline Drops | Verify that a counter with multiple drop reasons<br>still works correctly | SMAC_EQUALS_DMAC<br>SIP_LINK_LOCAL |
| 4.3.2 | Multiple Pipeline Drops | Verify that a packet that fulfills multiple drop<br>reasons is only counted once | SMAC_EQUALS_DMAC<br>SIP_LINK_LOCAL |
| 4.4.1 | Multiple Counters | Verify that two drop counters with different drop<br>reasons both count drops correctly | Counter 1:<br>SMAC_EQUALS_DMAC<br><br>Counter 2:<br>SIP_LINK_LOCAL |
| 4.4.2 | Multiple Counters | Verify that two drop counters with overlapping drop<br>reasons both count drops correctly | Counter 1:<br>SMAC_EQUALS_DMAC<br><br>Counter 2:<br>SMAC_EQUALS_DMAC<br>SIP_LINK_LOCAL |
| 4.5.1 | Configuration | Verify that a drop reason can be added to a counter<br>that has already been installed | SMAC_EQUALS_DMAC<br>SIP_LINK_LOCAL |
| 4.5.2 | Configuration | Verify that a drop reason can be removed from a counter<br>that has already been installed | SMAC_EQUALS_DMAC<br>SIP_LINK_LOCAL |
| 4.5.3 | Configuration | Verify that counter values are not retained after a<br>counter is deleted and re-added | SMAC_EQUALS_DMAC |
| 4.6.1 | Regressions | Verify that control plane packets are not captured<br>by the ACL_ANY counter | ACL_ANY |
| 4.6.2 | Regressions | Verify that invalid CONFIG_DB configs do not crash<br>the device | N/A |

### 1.3 Related CLI Commands
| Command | Usage |
|--------------------------------|---------------------------------|
| `show interfaces counters` | Check RX_DRP (SAI_*_STAT_IF_IN_DISCARDS) |
| `show dropcounters capabilities` | Check drop counter capabilities |
| `show dropcounters counts` | Check drop counter counts |
| `sonic-clear dropcounters` | Clear drop counters |

Note that `show dropcounters counts` may display different output depending on whether the DUT supports port-level and/or switch-level drop counters. This information will be gathered during the capabilities check and the output will be parsed accordingly depending on which types of counters are supported on the DUT.

For devices that support switch-level counters, the results from `show interfaces counters` will be aggregated for comparison against the switch-level drop counters.

## 2 Testing Parameters

### 2.1 Supported Topologies
This testbed currently only supports the `t0` topology. This may be expanded in the future.

### 2.2 Testbed Constraints
- Control plane traffic should be disabled unless explicitly required by a specific test case in order to simplify the testing logic
- All packets should be routable so that we can verify that packets are dropped and not forwarded

### 2.3 IPv6 Support
IPv4 as well as IPv6 should be tested for all L3 drop counters. For the sake of brevity, we will include sample IPv4 Sample Packets only.

## 3. Testing Flow

### 3.1 Setup
1. All control plane traffic should be disabled. We can do so by disabling the VMs from the testbed-cli utility.
2. Routes should be added so that we can monitor a subset of the interfaces for packets being forwarded.

This step can be done once before all test cases run.

### 3.2 Testing
1. Confirm that the capability being tested is supported by the DUT.
2. Clear interface counters on the DUT.
3. Setup relevant drop counters for the given test case.
4. Select ports to send traffic to and to sniff for outgoing traffic.
5. Send N = 10 packets to the DUT on the selected ports.
6. Verify that the selected ports have had both their standard drop counters and their configured drop counters incrememented by N.
7. Verify that non-selected ports are not affected.
8. Verify that traffic was not forwarded along any of the outgoing ports.
9. Delete all configured drop counters.

Note that certain tests may deviate from these steps - this will be noted in the test cases below.

### 3.3 Teardown
1. All control plane traffic should be re-enabled. We can do so by enabling the VMs from the testbed-cli utility.
2. Any routes that have been added should be cleaned up so that we do not interfere with subsequent test runs.

This step can be done once after all test cases have run.


# 4 Test Cases

## 4.1 Capabilities

### 4.1.1. Verify that drop counter capabilities can be queried
#### Counters: N/A

#### Packet Definitons: N/A

#### Additional Steps:
1. Confirm that orchagent and syncd are running.
2. Use `show dropcounters capabilities` to confirm that capabilities are available.
3. Use `redis-cli` to confirm that the capabilites returned by the CLI match what's stored in STATE DB.
4. Store supported capabilities for use by later tests.

## 4.2 Individual Pipeline Drops

### 4.2.1. Verify that SMAC_EQUALS_DMAC is counted correctly
#### Counters:
```
TEST_COUNTER
    Type: INGRESS
    Reasons: SMAC_EQUALS_DMAC
```

#### Sample Packets:
```
###[ Ethernet ]###
  dst = [DUT_MAC_ADDR]
  src = [DUT_MAC_ADDR]
  type = 0x800
...
```

#### Additional Steps: N/A

### 4.2.2. Verify that ACL_ANY is counted correctly
#### Counters:
```
TEST_COUNTER
    Type: INGRESS
    Reasons: ACL_ANY
```

#### Sample Packets:
Packet A:
```
###[ Ethernet ]###
  dst = [auto]
  src = [auto]
  type = 0x800
###[ IP ]###
    version = 4
    ttl = [auto]
    proto = tcp
    src = 10.0.0.1
    dst = 2.2.2.2
...
```

Packet B:
```
###[ Ethernet ]###
  dst = [auto]
  src = [auto]
  type = 0x800
###[ IP ]###
    version = 4
    ttl = [auto]
    proto = tcp
    src = 10.0.0.1
    dst = 3.3.3.3
...
```

#### Additional Steps:
For this test we need to configure ACL rules on the device as follows:
```
Rule 1:
    MATCH: DST IP = 2.2.2.2
    ACTION: DROP

Rule 2:
    MATCH: DST IP = 3.3.3.3
    ACTION: FORWARD
```

In addition to sending packets to be dropped, we will also send packets that will be forwarded in order to ensure that the drop counter is only counting _drops_, not just the number of packets that trigger ACL rules. These packets should not appear in the drop counters or RX_DRP, and they should be received at the configured egress port.

### 4.2.3. Verify that SIP_LINK_LOCAL is counted correctly
#### Counters:
```
TEST_COUNTER
    Type: INGRESS
    Reasons: SIP_LINK_LOCAL
```

#### Sample Packets:
```
###[ Ethernet ]###
  dst = [auto]
  src = [auto]
  type = 0x800
###[ IP ]###
    version = 4
    ttl = [auto]
    proto = tcp
    src = 169.254.0.0
    dst = 2.2.2.2
...
```

#### Additional Steps: N/A

### 4.2.4. Verify that DIP_LINK_LOCAL is counted correctly
#### Counters:
```
TEST_COUNTER
    Type: INGRESS
    Reasons: DIP_LINK_LOCAL
```

#### Sample Packets:
```
###[ Ethernet ]###
  dst = [auto]
  src = [auto]
  type = 0x800
###[ IP ]###
    version = 4
    ttl = [auto]
    proto = tcp
    src = 10.0.0.1
    dst = 169.254.0.0
...
```

#### Additional Steps: N/A

### 4.2.5. Verify that L3_EGRESS_LINK_DOWN is counted correctly
#### Counters:
```
TEST_COUNTER
    Type: INGRESS
    Reasons: L3_EGRESS_LINK_DOWN
```

#### Sample Packets:
```
###[ Ethernet ]###
  dst = [auto]
  src = [auto]
  type = 0x800
###[ IP ]###
    version = 4
    ttl = [auto]
    proto = tcp
    src = 10.0.0.1
    dst = 4.4.4.4
...
```

#### Additional Steps:
For this test we need to shutdown the interface that packets destined for 4.4.4.4 will be sent to, and bring it back up after the test has finished.

We should also send some packets to that interface before shutting it down to ensure that everything is forwarding correctly before we modify the system.

### 4.2.6. Verify that UNRESOLVED_NEXT_HOP is counted correctly
#### Counters:
```
TEST_COUNTER
    Type: INGRESS
    Reasons: UNRESOLVED_NEXT_HOP
```

#### Sample Packets:
```
###[ Ethernet ]###
  dst = [auto]
  src = [auto]
  type = 0x800
###[ IP ]###
    version = 4
    ttl = [auto]
    proto = tcp
    src = 10.0.0.1
    dst = 4.4.4.4
...
```

#### Additional Steps:
For this test we need to delete the route to 4.4.4.4 so that the next hop cannot be resolved. We will bring it back after the test has finished.

We should also send some packets to that interface before shutting it down to ensure that everything is forwarding correctly before we modify the system.

## 4.3 Multiple Pipeline Drops

### 4.3.1. Verify that a counter with multiple drop reasons still works correctly
#### Counters:
```
TEST_COUNTER
    Type: INGRESS
    Reasons: SMAC_EQUALS_DMAC, SIP_LINK_LOCAL
```

#### Sample Packets:
Packet A:
```
###[ Ethernet ]###
  dst = [DUT_MAC_ADDR]
  src = [DUT_MAC_ADDR]
  type = 0x800
...
```

Packet B:
```
###[ Ethernet ]###
  dst = [auto]
  src = [auto]
  type = 0x800
###[ IP ]###
    version = 4
    ttl = [auto]
    proto = tcp
    src = 169.254.0.0
    dst = 2.2.2.2
...
```

#### Additional Steps: N/A

### 4.3.2. Verify that a packet that fulfills multiple drop reasons is only counted once
#### Counters:
```
TEST_COUNTER
    Type: INGRESS
    Reasons: SMAC_EQUALS_DMAC, SIP_LINK_LOCAL
```

#### Sample Packets:
```
###[ Ethernet ]###
  dst = [DUT_MAC_ADDR]
  src = [DUT_MAC_ADDR]
  type = 0x800
###[ IP ]###
    version = 4
    ttl = [auto]
    proto = tcp
    src = 169.254.0.0
    dst = 2.2.2.2
...
```

#### Additional Steps: N/A

## 4.4 Multiple Counters

### 4.4.1. Verify that two drop counters with different drop reasons both count drops correctly
#### Counters:
```
TEST_COUNTER_0
    Type: INGRESS
    Reasons: SMAC_EQUALS_DMAC

TEST_COUNTER_1
    Type: INGRESS
    Reasons: SIP_LINK_LOCAL
```

#### Sample Packets:
Packet A:
```
###[ Ethernet ]###
  dst = [DUT_MAC_ADDR]
  src = [DUT_MAC_ADDR]
  type = 0x800
...
```

Packet B:
```
###[ Ethernet ]###
  dst = [auto]
  src = [auto]
  type = 0x800
###[ IP ]###
    version = 4
    ttl = [auto]
    proto = tcp
    src = 169.254.0.0
    dst = 2.2.2.2
...
```

#### Additional Steps:
Because the counters are non-overlapping, the sum of the two counters should yield the total number of drops when we perform the calculation.

### 4.4.2. Verify that two drop counters with overlapping drop reasons both count drops correctly
#### Counters:
```
TEST_COUNTER_0
    Type: INGRESS
    Reasons: SMAC_EQUALS_DMAC

TEST_COUNTER_1
    Type: INGRESS
    Reasons: SMAC_EQUALS_DMAC, SIP_LINK_LOCAL
```

#### Sample Packets:
Packet A:
```
###[ Ethernet ]###
  dst = [DUT_MAC_ADDR]
  src = [DUT_MAC_ADDR]
  type = 0x800
...
```

Packet B:
```
###[ Ethernet ]###
  dst = [auto]
  src = [auto]
  type = 0x800
###[ IP ]###
    version = 4
    ttl = [auto]
    proto = tcp
    src = 169.254.0.0
    dst = 2.2.2.2
...
```

#### Additional Steps:
Because counter B includes both drop reasons, the total number of drops counted by counter B should equal the total number of packets sent for this test case. Counter A should only be equal to the number of SMAC_EQUALS_DMAC packets sent.

**NOTE:** This test case is not supported on Mellanox devices as their counters do not allow overlapping drop reasons.

## 4.5 Configuration

### 4.5.1. Verify that a drop reason can be added to a counter that has already been installed
#### Counters:
```
TEST_COUNTER
    Type: INGRESS
    Reasons: SMAC_EQUALS_DMAC -> SMAC_EQUALS_DMAC, SIP_LINK_LOCAL
```

#### Sample Packets:
Packet A:
```
###[ Ethernet ]###
  dst = [DUT_MAC_ADDR]
  src = [DUT_MAC_ADDR]
  type = 0x800
...
```

Packet B:
```
###[ Ethernet ]###
  dst = [auto]
  src = [auto]
  type = 0x800
###[ IP ]###
    version = 4
    ttl = [auto]
    proto = tcp
    src = 169.254.0.0
    dst = 2.2.2.2
...
```

#### Additional Steps:
We essentially want to repeat the general testing procedure twice for this test. The procedure is as follows:
1. Send SMAC_EQUALS_DMAC packets to the device, verify that the counts are correct.
2. Add SIP_LINK_LOCAL as a drop reason to the counter, verify that it is cleared.
3. Send both SMAC_EQUALS_DMAC and SIP_LINK_LOCAL packets to the device, verify that both are counted by TEST_COUNTER.

### 4.5.2. Verify that a drop reason can be removed from a counter that has already been installed
#### Counters:
```
TEST_COUNTER
    Type: INGRESS
    Reasons: SMAC_EQUALS_DMAC, SIP_LINK_LOCAL -> SMAC_EQUALS_DMAC
```

#### Sample Packets:
Packet A:
```
###[ Ethernet ]###
  dst = [DUT_MAC_ADDR]
  src = [DUT_MAC_ADDR]
  type = 0x800
...
```

Packet B:
```
###[ Ethernet ]###
  dst = [auto]
  src = [auto]
  type = 0x800
###[ IP ]###
    version = 4
    ttl = [auto]
    proto = tcp
    src = 169.254.0.0
    dst = 2.2.2.2
...
```

#### Additional Steps:
We essentially want to repeat the general testing procedure twice for this test. The procedure is as follows:
1. Send both SMAC_EQUALS_DMAC and SIP_LINK_LOCAL packets to the device, verify that the counts are correct.
2. Remove SIP_LINK_LOCAL as a drop reason from the counter, verify that it is cleared.
3. Send both SMAC_EQUALS_DMAC and SIP_LINK_LOCAL packets to the device, verify that both are counted by RX_DRP but only SMAC_EQUALS_DMAC is counted by TEST_COUNTER.

### 4.5.3. Verify that counter values are not retained after a counter is deleted and re-added
#### Counters:
```
TEST_COUNTER
    Type: INGRESS
    Reasons: SMAC_EQUALS_DMAC
```

#### Sample Packets:
```
###[ Ethernet ]###
  dst = [DUT_MAC_ADDR]
  src = [DUT_MAC_ADDR]
  type = 0x800
...
```

#### Additional Steps:
1. Run through the general testing procedure as normal.
2. Re-create TEST_COUNTER. Verify that the count is 0.
3. Delete TEST_COUNTER.

## 4.6 Regressions

### 4.6.1 Verify that control plane packets are not captured by the ACL_ANY counter
#### Counters:
```
TEST_COUNTER
    Type: INGRESS
    Reasons: ACL_ANY
```

#### Sample Packets: N/A

#### Additional Steps:
For this test, we just want to wait 5-10s and verify that none of the packet drop counters are affected by LAGs or BGP traffic.

We will need to enable the VMs for this test in order to generate control plane traffic, and we will need to disable them after we are done.

### 4.6.2 Verify that invalid CONFIG_DB configs do not crash the device
#### Counters:
```
BAD_TYPE:
    Type: BAD_TYPE
    Reasons: ACL_ANY

BAD_REASONS:
    Type: Ingress
    Reasons: FOO, BAR, BAZ
```

#### Sample Packets: N/A

#### Additional Steps:
For this test, we will apply the drop counter configurations directly to CONFIG DB. No counters should be created, and orchagent and syncd should not crash.

Additionally, we can use the log analyzer to validate that DebugCounterOrch is correctly emitting errors (but not crashing!).

## 5 Future Improvements
- Add addtional tests for additional drop reasons. At the moment we're only testing a few different drop types because vendor support for this feature is very limited. As adoption grows we will add tests for new drop reasons.
- Another thing that might be useful is having "templated" tests that use a YAML file or something similar to describe what counters and reasons the user wants to test. This would be more flexible than what we currently have defined, and would allow people to setup the test to validate the counter configurations that they are using in production, in addition to the pre-defined tests.

## 6 Acknowledgements
Thank you to Prince and Guohan from Microsoft as well as Yuriy and Liat from Mellanox for their help coming up with this test setup and the test cases.

## 7 References
[1] [SONiC Test Ingress Discards HLD](https://github.com/Azure/SONiC/pull/514)