# CPU queue stats

Network switches and routers typically have one port connected to the CPU (where the control software runs). This port is referenced as a CPU port on this document. Multiple egress queues on the CPU port helps differentiate traffic and avoid one kind of traffic overwhelming the other. Switch control software installs rules that identify control protocol packets and trap them to a particular queue on the CPU port. Please refer to COPP for more details.

The knowledge of various queue statistics help us identify the current health and reason for any packet drops. SONiC already has commands to examine port and queue statistics associated with front panel ports. The same is extended to examine the CPU queue stats. The watermark commands can also be extended to display the buffer utilization associated with these CPU queues (not included in this HLD).

The command *show queue counters <port-name>* displays the below given statistics for a given front panel port.

```
root@sonic:/home/admin# show queue counters Ethernet0
     Port    TxQ    Counter/pkts    Counter/bytes    Drop/pkts    Drop/bytes
---------  -----  --------------  ---------------  -----------  ------------
Ethernet0    UC0               0                0            0             0
Ethernet0    UC1               0                0            0             0
Ethernet0    UC2               0                0            0             0
Ethernet0    UC3               0                0            0             0
Ethernet0    UC4               0                0            0             0
Ethernet0    UC5               0                0            0             0
Ethernet0    UC6               0                0            0             0
Ethernet0    UC7               0                0            0             0
Ethernet0    UC8               0                0            0             0
Ethernet0    UC9               0                0            0             0
Ethernet0   MC10               0                0            0             0
Ethernet0   MC11               0                0            0             0
Ethernet0   MC12               0                0            0             0
Ethernet0   MC13               0                0            0             0
Ethernet0   MC14               0                0            0             0
Ethernet0   MC15               0                0            0             0
Ethernet0   MC16               0                0            0             0
Ethernet0   MC17               0                0            0             0
Ethernet0   MC18               0                0            0             0
Ethernet0   MC19               0                0            0             0
```

On current SONiC code base, the command returns *Port doesn't exist! CPU* as the CPU port is not understood by SONiC.

```
root@sonic:/home/admin# show queue counters CPU
Port doesn't exist! CPU
```

The proposed changes make the command *show queue counters CPU* return the actual supported CPU queues and its stats.

```
root@L15:/home/admin# show queue counters CPU
  Port    TxQ    Counter/pkts    Counter/bytes    Drop/pkts    Drop/bytes
------  -----  --------------  ---------------  -----------  ------------
   CPU    MC0               0                0            0             0
   CPU    MC1               0                0            0             0
   CPU    MC2               0                0            0             0
   CPU    MC3               0                0            0             0
   CPU    MC4               0                0            0             0
   CPU    MC5               0                0            0             0
   CPU    MC6               0                0            0             0
   CPU    MC7               0                0            0             0
   CPU    MC8              14             1484            0             0
   CPU    MC9             175            63092            0             0
   CPU   MC10            1810           123080            0             0
   CPU   MC11               0                0            0             0
   CPU   MC12               0                0            0             0
   CPU   MC13               0                0            0             0
   CPU   MC14               0                0            0             0
   CPU   MC15               0                0            0             0
   CPU   MC16           75778          6470474            0             0
   CPU   MC17               0                0            0             0
   CPU   MC18           21837          6601410            0             0
   CPU   MC19               0                0            0             0
   CPU   MC20               0                0            0             0
   CPU   MC21               0                0            0             0
   CPU   MC22               0                0            0             0
   CPU   MC23              76            10032            0             0
   CPU   MC24               0                0            0             0
   CPU   MC25               0                0            0             0
   CPU   MC26               0                0            0             0
   CPU   MC27               0                0            0             0
   CPU   MC28               0                0            0             0
   CPU   MC29               0                0            0             0
   CPU   MC30               0                0            0             0
   CPU   MC31               0                0            0             0
   CPU   MC32               0                0            0             0
   CPU   MC33               0                0            0             0
   CPU   MC34               0                0            0             0
   CPU   MC35               0                0            0             0
   CPU   MC36               0                0            0             0
   CPU   MC37               0                0            0             0
   CPU   MC38               0                0            0             0
   CPU   MC39               0                0            0             0
   CPU   MC40               0                0            0             0
   CPU   MC41               0                0            0             0
   CPU   MC42               0                0            0             0
   CPU   MC43               0                0            0             0
   CPU   MC44               0                0            0             0
   CPU   MC45               0                0            0             0
   CPU   MC46               0                0            0             0
   CPU   MC47               0                0            0             0
```


SONiC DB changes:   
The following COUNTER_DB maps are changed to include details for the CPU port and its queues.   
```
COUNTERS_PORT_NAME_MAP   
COUNTERS_QUEUE_NAME_MAP   
COUNTERS_QUEUE_PORT_MAP   
COUNTERS_QUEUE_TYPE_MAP   
COUNTERS_QUEUE_INDEX_MAP   
```

Port and queue counters:   
```
COUNTERS:oid:<portOid>   
COUNTERS:oid:<queueOid>   
```

Code changes:   
Minor code changes are made on *sonic-swss* repo to fetch the supported queues on CPU port and update the flex counters-db maps to include CPU port and its queues.   
Single line changes are made to *sonic-py-swsssdk*, *sonic-snmpagent* and *sonic-utilities* repos to ignore the newly added CPU port while iterating over the front panel ports(*COUNTERS_PORT_NAME_MAP*).   
*sonic-sairedis* repo changes are made to enable cpu queues on vslib.    

SAI calls:   
SAI API *get_port_attribute()* with attribute *SAI_PORT_ATTR_QOS_NUMBER_OF_QUEUES* is called to fetch the number of CPU queues supported by switch.   
SAI API *get_port_attribute()* with attribute *SAI_PORT_ATTR_QOS_QUEUE_LIST* to fetch the CPU queue list.   

Synd calls SAI API *getStats()* with attributes *SAI_OBJECT_TYPE_PORT* and *SAI_OBJECT_TYPE_QUEUE* to fetch the stats.   

Handling SAI implementations without CPU queue support:   
Existing PortsOrch::initializeQueues() API throws *runtime_error("PortsOrch initialization failure.")* when above mentioned attributes are not supported. The code can be made to handle/ignore this exception for CPU queues.

Flex counters:   
Along with other flex counter stats, Syncd periodically fetches the CPU port and queue stats to update the counters-DB.   

Flex Counter attributes:
```
const vector<sai_port_stat_t> port_stat_ids =
{
    SAI_PORT_STAT_IF_IN_OCTETS,
    SAI_PORT_STAT_IF_IN_UCAST_PKTS,
    SAI_PORT_STAT_IF_IN_NON_UCAST_PKTS,
    SAI_PORT_STAT_IF_IN_DISCARDS,
    SAI_PORT_STAT_IF_IN_ERRORS,
    SAI_PORT_STAT_IF_IN_UNKNOWN_PROTOS,
    SAI_PORT_STAT_IF_OUT_OCTETS,
    SAI_PORT_STAT_IF_OUT_UCAST_PKTS,
    SAI_PORT_STAT_IF_OUT_NON_UCAST_PKTS,
    SAI_PORT_STAT_IF_OUT_DISCARDS,
    SAI_PORT_STAT_IF_OUT_ERRORS,
    SAI_PORT_STAT_IF_OUT_QLEN,
    SAI_PORT_STAT_IF_IN_MULTICAST_PKTS,
    SAI_PORT_STAT_IF_IN_BROADCAST_PKTS,
    SAI_PORT_STAT_IF_OUT_MULTICAST_PKTS,
    SAI_PORT_STAT_IF_OUT_BROADCAST_PKTS,
    SAI_PORT_STAT_ETHER_RX_OVERSIZE_PKTS,
    SAI_PORT_STAT_ETHER_TX_OVERSIZE_PKTS,
    SAI_PORT_STAT_PFC_0_TX_PKTS,
    SAI_PORT_STAT_PFC_1_TX_PKTS,
    SAI_PORT_STAT_PFC_2_TX_PKTS,
    SAI_PORT_STAT_PFC_3_TX_PKTS,
    SAI_PORT_STAT_PFC_4_TX_PKTS,
    SAI_PORT_STAT_PFC_5_TX_PKTS,
    SAI_PORT_STAT_PFC_6_TX_PKTS,
    SAI_PORT_STAT_PFC_7_TX_PKTS,
    SAI_PORT_STAT_PFC_0_RX_PKTS,
    SAI_PORT_STAT_PFC_1_RX_PKTS,
    SAI_PORT_STAT_PFC_2_RX_PKTS,
    SAI_PORT_STAT_PFC_3_RX_PKTS,
    SAI_PORT_STAT_PFC_4_RX_PKTS,
    SAI_PORT_STAT_PFC_5_RX_PKTS,
    SAI_PORT_STAT_PFC_6_RX_PKTS,
    SAI_PORT_STAT_PFC_7_RX_PKTS,
    SAI_PORT_STAT_PAUSE_RX_PKTS,
    SAI_PORT_STAT_PAUSE_TX_PKTS,
    SAI_PORT_STAT_ETHER_STATS_TX_NO_ERRORS,
    SAI_PORT_STAT_IP_IN_UCAST_PKTS,
    SAI_PORT_STAT_ETHER_IN_PKTS_128_TO_255_OCTETS,
};


static const vector<sai_queue_stat_t> queue_stat_ids =
{
    SAI_QUEUE_STAT_PACKETS,
    SAI_QUEUE_STAT_BYTES,
    SAI_QUEUE_STAT_DROPPED_PACKETS,
    SAI_QUEUE_STAT_DROPPED_BYTES,
};

static const vector<sai_queue_stat_t> queueWatermarkStatIds =
{
    SAI_QUEUE_STAT_SHARED_WATERMARK_BYTES,
};
```

List of PRs associated with this change:   
https://github.com/Azure/sonic-swss/pull/1544   
https://github.com/Azure/sonic-py-swsssdk/pull/98   
https://github.com/Azure/sonic-utilities/pull/1314   
https://github.com/Azure/sonic-snmpagent/pull/182   
https://github.com/Azure/sonic-sairedis/pull/732     


Possible future additions (not included in above PR list):      
```
root@sonic:/home/admin# show queue watermark CPU
Egress shared pool occupancy per CPU queue
  Queue    Bytes
-------  -------
  CPU:0        0
  CPU:1        0
  CPU:2        0
  CPU:3        0
  CPU:4        0
  CPU:5        0
  CPU:6        0
  CPU:7        0
  CPU:8        0
  CPU:9        0
 CPU:10        0
 CPU:11        0
 CPU:12        0
 CPU:13        0
 CPU:14        0
 CPU:15        0
 CPU:16        0
 CPU:17        0
 CPU:18      512
 CPU:19        0
 CPU:20        0
 CPU:21        0
 CPU:22        0
 CPU:23     1792
 CPU:24        0
 CPU:25        0
 CPU:26        0
 CPU:27        0
 CPU:28        0
 CPU:29        0
 CPU:30        0
 CPU:31        0
 CPU:32        0
 CPU:33        0
 CPU:34        0
 CPU:35        0
 CPU:36        0
 CPU:37        0
 CPU:38        0
 CPU:39        0
 CPU:40        0
 CPU:41        0
 CPU:42        0
 CPU:43        0
 CPU:44        0
 CPU:45        0
 CPU:46        0
 CPU:47        0
 ```
