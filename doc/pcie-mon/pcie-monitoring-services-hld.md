# SONiC PCIe Monitoring services HLD #

### Rev 0.5 ###

### Revision
 | Rev |     Date    |            Author            | Change Description                             |
 |:---:|:-----------:|:----------------------------:|------------------------------------------------|
 | 0.1 |             |     Sujin Kang               | Initial version                                |
 | 0.2 |             |     Sujin Kang               | Add rescan for pcie device missing during boot |
 |     |             |                              | Add pcied to PMON for runtime monitoring       |
 | 0.3 |             | Arun Saravanan Balachandran  | Add AER stats update support in pcied          |
 |     |             |                              | Add command to display AER stats               |
 | 0.4 |             | Arun Saravanan Balachandran  | Add platform API to collect AER stats          |
 | 0.5 |             | Arun Saravanan Balachandran  | Add options for pcie-aer sub-commands          |

## About This Manual ##

This document is intended to give the idea of how to monitor the platform PCIe devices and alert of any problems on PCIe buses and devices on SONiC using pcie-check service and pcied on PMON container.


## 1. PCIe Monitor service design ##

New PCIe Monitor service is designed to use the PcieUtil utility to check the current status of PCIe devices and buses and alert if there is any missing devices or any error while communicating on the PCIe buses.

PCIe device monitoring will be done in two separate services, `pcie-check.service` which is a systemd service, will check the PCIe device during the boot time and `pcied` which is a daemon in PMON container will monitor during the runtime.

First, pcie-check.service will be added to check the pcie device enumeration status, trigger a retry of a pci device rescan if there is any missing device and save the result status of pcie device check into the STATE_DB to indicate any device missing to the party that are interested in the device enumeration, for example, kernel_bde driver, platform drivers and etc.

Second, pcied in PMON will perform the periodic pcie device check during the run time.

Both pcie-check.service and pcied will update the state db with the PCIe device status whenever it changes.

### 1.1 Access the PCIe devices and buses from platform ###

PCIe device information can be accessed via read files under (e.g. `/sys/bus/pci/devices/0000:01:00.1`), different vendors may have under different folders, these folder need to be mounted to platform container so pcied can access them. 

For the convenience of implementation and reduce the time consuming, pcie-check.service will use the `pcieutil` which is the pcie diag tool. `pcieutil` is implemented based on platform_base.sonic_pcie.`PcieUtil` class.

1. `pcieutil` should get the platform specific PCIe device information and monitor the PCIe device and bus status with PcieUtil.get_pcie_check.

2. `PcieUtil` will provide APIs `load_config_file`, `get_pcie_device` and `get_pcie_check` to get the expected PCIe device list and informations, to get the current PCIe device information, and check if any PCIe device is missing or if there is any PCIe bus error.

![pcieinfo_design](https://github.com/Azure/SONiC/blob/master/doc/pcieinfo_design.md)

### 1.2 PCIe device configuration file ###

PcieUtil needs to get the expected PCIe device information to check the PCIe device status periodically, which is different for each platform/hardware sku.

Each vendor need to generate the PCIe device configuration file name as pcie.yml and locate the file under `device/<platform>/<hardware_skus>/plugins`. 

Example) Location: `device/celestica/x86_64-cel_seastone-r0/plugins/pcie.yaml`

```
...
- bus: '01'
  dev: '00'
  fn: '0'
  id: b960
  name: 'Ethernet controller: Broadcom Limited Broadcom BCM56960 Switch ASIC'
- bus: '01'
  dev: '00'
  fn: '1'
  id: b960
  name: 'Ethernet controller: Broadcom Limited Broadcom BCM56960 Switch ASIC'
```

### 1.3 PCIe device status check ###


The default PCIe device check function, get_pcie_check is implemented in PcieUtil class at sonic_platform_base/sonic_pcie/pcie_common.py.
It loads the PCIe device configuration file and compares them with the enumerated devices based on the platform sysfs device tree under /sys/bus/pci/devices/.

Here we define a common platform API to in class `PcieBase`: 

    @abc.abstractmethod
    def get_pcie_check(self, timeout=0):
        """
         Check Pcie device with config file
         Returns:
            A list including pcie device and test result info
        """
        return []

Each vendor need to implement this function in `PcieBase` plugin if vendor has any additional pcie healthy check method.

PcieUtil calls this API to check the PCIe device status, following example code showing how this API will be called:

    while True:
        status, device_dict = platform_pcieutil.get_pcie_check()
        if(status):
            for key, value in device_dict.iteritems():
                print("Device on PCIe bus: %s" was %s" % (key, value))
                 
### 1.4 PCIe Check Service `pcie-check.service` flow ###

pcie-check.service will be started by systemd during boot up and it will spawn a thread to check PCIe device status and perform the rescan pci devices if there is any missing devices after rc.local.service is completed and it will update the state db with pcie device satus after the `pcieutil pcie-chek` call so that the dependent services/container or kernel driver can be started or stopped based on the status.

Detailed flow as showed in below chart: 
![](https://github.com/Azure/SONiC/blob/70a152f1b98e145c9f0771e7cda7a951d98a978e/images/pcie-check.svg)


### 1.5 PCIe daemon `pcied` flow ###

pcied will be started by PMON container will continue monitoring the PCIe device status during run time and it will check the PCIe device status periodically every 1 minute and update the state db when the status is checked.

Detailed flow as showed in below chart:
![](https://github.com/Azure/SONiC/blob/70a152f1b98e145c9f0771e7cda7a951d98a978e/images/pcied.svg)


### 1.6 STATE_DB keys and value ###

The PCIe Monitoring services, pcie-check.service and pcied update the STATE_DB when they check the PCIe device status. The keys for PCIe device STATUS are "PCIE_STATUS|PCIE_DEVICES" with values of "PASSED" and "FAILED".
```
user@server:~$ redis-cli -n 6 SET "PCIE_STATUS|PCIE_DEVICES" "PASSED"
OK
user@server:~$ redis-cli -n 6 SET "PCIE_STATUS|PCIE_DEVICES"
"PASSED"
```

## 2. PCIe AER stats collection design ##

The PCIe AER stats for the supported PCIe devices will be collected by `pcied` and updated in STATE_DB during runtime.

New sub-command group `pcie-aer` will be implemented in `pcieutil` to retrieve and tabulate the PCIe AER stats from STATE_DB.

### 2.1 Access the PCIe devices' AER stats from platform ###

For AER supported PCIe device, the AER stats belonging to severities `correctable`, `fatal`, `non_fatal` can be accessed via files  (e.g. `/sys/bus/pci/devices/0000:01:00.1/aer_dev_correctable`, `/sys/bus/pci/devices/0000:01:00.1/aer_dev_fatal`, `/sys/bus/pci/devices/0000:01:00.1/aer_dev_nonfatal`) respectively.

### 2.2 PCIe AER stats collection in pcied ###

A common platform API `get_pcie_aer_stats` is defined in class `PcieBase` for retrieving AER stats of a PCIe device:

```
    @abc.abstractmethod
    def get_pcie_aer_stats(self, domain, bus, dev, fn):
        """
        Returns a nested dictionary containing the AER stats belonging to a
        PCIe device

        Args:
            domain, bus, dev, fn: Domain, bus, device, function of the PCIe
            device respectively

        Returns:
            A nested dictionary where key is severity 'correctable', 'fatal' or
            'non_fatal', value is a dictionary of key, value pairs in the format:
                {'AER Error type': Error count}

            Ex. {'correctable': {'BadDLLP': 0, 'BadTLP': 0},
                 'fatal': {'RxOF': 0, 'MalfTLP': 0},
                 'non_fatal': {'RxOF': 0, 'MalfTLP': 0}}

            For PCIe devices that do not support AER, the value for each severity
            key is an empty dictionary.
        """
        return {}
```

Default `get_pcie_aer_stats`is implemented in PcieUtil class at sonic_platform_base/sonic_pcie/pcie_common.py.
It returns the AER stats for a given PCIe device obtained from the AER sysfs under `/sys/bus/pci/devices/<Domain>:<Bus>:<Dev>.<Fn>`

For PCIe devices that pass PcieUtil `get_pcie_check`, AER stats will be retrieved using `get_pcie_aer_stats` and updated in the STATE_DB periodically every minute by pcied.

### 2.3 STATE_DB keys and value ###

The key used to represent a PCIE device for storing its attributes in STATE_DB is of the format `PCIE_DEVICE|<Bus>:<Dev>.<Fn>`.
For every device, AER stats will be stored as key, value pairs where key is of the format `<severity>|<AER Error type>` and the device ID will be stored with key `id`.

Example) For a PCIe device with Bus: 1, Dev: 0, Fn: 1, Id: b960 the STATE_DB entry will be as below:

```
"PCIE_DEVICE|01:00.0": {
  "expireat": 1607061625.1506171,
  "ttl": -0.001,
  "type": "hash",
  "value": {
    "correctable|BadDLLP": "0",
    "correctable|BadTLP": "2",
    "correctable|CorrIntErr": "0",
    "correctable|HeaderOF": "0",
    "correctable|NonFatalErr": "0",
    "correctable|Rollover": "0",
    "correctable|RxErr": "0",
    "correctable|TOTAL_ERR_COR": "2",
    "correctable|Timeout": "0",
    "fatal|ACSViol": "0",
    "fatal|AtomicOpBlocked": "0",
    "fatal|BlockedTLP": "0",
    "fatal|CmpltAbrt": "0",
    "fatal|CmpltTO": "0",
    "fatal|DLP": "0",
    "fatal|ECRC": "0",
    "fatal|FCP": "0",
    "fatal|MalfTLP": "0",
    "fatal|RxOF": "0",
    "fatal|SDES": "0",
    "fatal|TLP": "0",
    "fatal|TLPBlockedErr": "0",
    "fatal|TOTAL_ERR_FATAL": "0",
    "fatal|UncorrIntErr": "0",
    "fatal|Undefined": "0",
    "fatal|UnsupReq": "0",
    "fatal|UnxCmplt": "0",
    "id": "0xb960",
    "non_fatal|ACSViol": "0",
    "non_fatal|AtomicOpBlocked": "0",
    "non_fatal|BlockedTLP": "0",
    "non_fatal|CmpltAbrt": "0",
    "non_fatal|CmpltTO": "0",
    "non_fatal|DLP": "0",
    "non_fatal|ECRC": "0",
    "non_fatal|FCP": "0",
    "non_fatal|MalfTLP": "0",
    "non_fatal|RxOF": "0",
    "non_fatal|SDES": "0",
    "non_fatal|TLP": "0",
    "non_fatal|TLPBlockedErr": "0",
    "non_fatal|TOTAL_ERR_NONFATAL": "3",
    "non_fatal|UncorrIntErr": "0",
    "non_fatal|Undefined": "0",
    "non_fatal|UnsupReq": "3",
    "non_fatal|UnxCmplt": "0"
  }
}
```

### 2.4 PCIe AER stats CLI ###

Add a new "pcieutil pcie-aer" command line to display the AER stats.

```
root@sonic:/home/admin# pcieutil
Usage: pcieutil [OPTIONS] COMMAND [ARGS]...

  pcieutil - Command line utility for checking pci device

Options:
  --help  Show this message and exit.

Commands:
  pcie-aer       Display PCIe AER status
  pcie-check     Check PCIe Device
  pcie-generate  Generate config file with current pci device
  pcie-show      Display PCIe Device
  version        Display version info
root@sonic:/home/admin#
```

"pcieutil pcie-aer" has four sub commands 'all', 'correctable', 'fatal' and 'non-fatal'.
'all' command displays the AER stats for all severities. 'correctable', 'fatal' and 'non-fatal' commands display the AER stats of respective severity.

```
root@sonic:/home/admin# pcieutil pcie-aer
Usage: pcieutil pcie-aer [OPTIONS] COMMAND [ARGS]...

  Display PCIe AER status

Options:
  --help  Show this message and exit.

Commands:
  all          Show all PCIe AER attributes
  correctable  Show correctable PCIe AER attributes
  fatal        Show fatal PCIe AER attributes
  non-fatal    Show non-fatal PCIe AER attributes
root@sonic:/home/admin#
```

Each "pcie-aer" sub command displays only non-zero AER stats by default and has below options:
- `-d/--device <Bus>:<Dev>.<Fn>` - Display stats only for the specified device
- `-v/--verbose` - Display all stats

```
root@sonic:/home/admin# pcieutil pcie-aer all --help
Usage: pcieutil pcie-aer all [OPTIONS]

  Show all PCIe AER attributes
  (Default: Display only non-zero attributes)

Options:
  -d, --device <BUS>:<DEV>.<FN>  Display stats only for the specified device
  -v, --verbose                  Display all stats
  --help                         Show this message and exit.
root@sonic:/home/admin#
```

Sample output:

```
root@sonic:/home/admin# pcieutil pcie-aer all
+---------------------+-----------+-----------+
| AER - CORRECTABLE   |   01:00.0 |   01:00.1 |
|                     |    0xb960 |    0xb960 |
+=====================+===========+===========+
| BadTLP              |         2 |         2 |
+---------------------+-----------+-----------+
| BadDLLP             |         3 |         0 |
+---------------------+-----------+-----------+
| TOTAL_ERR_COR       |         5 |         2 |
+---------------------+-----------+-----------+

+--------------------+-----------+
| AER - NONFATAL     |   01:00.1 |
|                    |    0xb960 |
+====================+===========+
| UnsupReq           |         3 |
+--------------------+-----------+
| TOTAL_ERR_NONFATAL |         3 |
+--------------------+-----------+

root@sonic:/home/admin#
root@sonic:/home/admin# pcieutil pcie-aer all -v
+---------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| AER - CORRECTABLE   |   00:01.0 |   00:02.0 |   00:03.0 |   00:04.0 |   00:0f.0 |   00:13.0 |   00:14.0 |   00:14.1 |   00:14.2 |   01:00.0 |   01:00.1 |
|                     |    0x1f10 |    0x1f11 |    0x1f12 |    0x1f13 |    0x1f16 |    0x1f15 |    0x1f41 |    0x1f41 |    0x1f41 |    0xb960 |    0xb960 |
+=====================+===========+===========+===========+===========+===========+===========+===========+===========+===========+===========+===========+
| RxErr               |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+---------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| BadTLP              |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         2 |         2 |
+---------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| BadDLLP             |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         3 |         0 |
+---------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| Rollover            |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+---------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| Timeout             |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+---------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| NonFatalErr         |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+---------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| CorrIntErr          |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+---------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| HeaderOF            |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+---------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| TOTAL_ERR_COR       |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         5 |         2 |
+---------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+

+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| AER - FATAL     |   00:01.0 |   00:02.0 |   00:03.0 |   00:04.0 |   00:0f.0 |   00:13.0 |   00:14.0 |   00:14.1 |   00:14.2 |   01:00.0 |   01:00.1 |
|                 |    0x1f10 |    0x1f11 |    0x1f12 |    0x1f13 |    0x1f16 |    0x1f15 |    0x1f41 |    0x1f41 |    0x1f41 |    0xb960 |    0xb960 |
+=================+===========+===========+===========+===========+===========+===========+===========+===========+===========+===========+===========+
| Undefined       |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| DLP             |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| SDES            |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| TLP             |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| FCP             |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| CmpltTO         |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| CmpltAbrt       |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| UnxCmplt        |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| RxOF            |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| MalfTLP         |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| ECRC            |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| UnsupReq        |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| ACSViol         |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| UncorrIntErr    |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| BlockedTLP      |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| AtomicOpBlocked |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| TLPBlockedErr   |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| TOTAL_ERR_FATAL |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+-----------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+

+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| AER - NONFATAL     |   00:01.0 |   00:02.0 |   00:03.0 |   00:04.0 |   00:0f.0 |   00:13.0 |   00:14.0 |   00:14.1 |   00:14.2 |   01:00.0 |   01:00.1 |
|                    |    0x1f10 |    0x1f11 |    0x1f12 |    0x1f13 |    0x1f16 |    0x1f15 |    0x1f41 |    0x1f41 |    0x1f41 |    0xb960 |    0xb960 |
+====================+===========+===========+===========+===========+===========+===========+===========+===========+===========+===========+===========+
| Undefined          |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| DLP                |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| SDES               |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| TLP                |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| FCP                |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| CmpltTO            |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| CmpltAbrt          |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| UnxCmplt           |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| RxOF               |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| MalfTLP            |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| ECRC               |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| UnsupReq           |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         3 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| ACSViol            |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| UncorrIntErr       |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| BlockedTLP         |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| AtomicOpBlocked    |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| TLPBlockedErr      |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
| TOTAL_ERR_NONFATAL |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         0 |         3 |
+--------------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+

root@sonic:/home/admin#
root@sonic:/home/admin# pcieutil pcie-aer correctable -d 00:01.0 -v
+---------------------+-----------+
| AER - CORRECTABLE   |   00:01.0 |
|                     |    0x1f10 |
+=====================+===========+
| RxErr               |         0 |
+---------------------+-----------+
| BadTLP              |         0 |
+---------------------+-----------+
| BadDLLP             |         0 |
+---------------------+-----------+
| Rollover            |         0 |
+---------------------+-----------+
| Timeout             |         0 |
+---------------------+-----------+
| NonFatalErr         |         0 |
+---------------------+-----------+
| CorrIntErr          |         0 |
+---------------------+-----------+
| HeaderOF            |         0 |
+---------------------+-----------+
| TOTAL_ERR_COR       |         0 |
+---------------------+-----------+
root@sonic:/home/admin#
```


< TBA >
## Open Questions ##

1. Current PcieUtil is limited to check the PCIe device availablility based on the configuration. 
   Can we also add the PCIe communication error status check using AER detection into get_pcie_check() api or with a separate api? 
   some plugins like, say, collectd (https://wiki.opnfv.org/display/fastpath/PCIe+Advanced+Error+Reporting+Plugin)  
