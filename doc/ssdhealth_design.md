## Motivation
Add to SONiC an ability to check storage health state. Basic functionality will be implemented as a CLI command. Optionally pmon daemon could be added for constant disk state monitoring.

## CLI

### Syntax
    show platform ssdhealth [verbose/vendor]

### Output example
#### Brief
    admin@sonic-switch: ~$ show platform ssdhealth
    Device Model : InnoDisk Corp. - mSATA 3ME
    Health: 72.9%
    Temperature: N/A
    admin@sonic-switch: ~$

#### Verbose
    admin@sonic-switch: ~$ show platform ssdhealth verbose
    Device Model     : InnoDisk Corp. - mSATA 3ME
    FW Version       : S140714
    Serial Number    : 20160429AA1134000035
    Health           : 72.9%
    Capacity         : 29.818199 GB
    Temperature      : N/A
    Power On Hours   : 1576 hours
    Power Cycle count: 130
    Something else???

#### Vendor
    admin@sonic-switch: ~$ show platform ssdhealth vendor

	********************************************************************************************
	* Innodisk iSMART V3.9.41                                                       2018/05/25 *
	********************************************************************************************
	Model Name: InnoDisk Corp. - mSATA 3ME
	FW Version: S140714
	Serial Number: 20160429AA1134000035
	Health: 72.900%
	Capacity: 29.818199 GB
	P/E Cycle: 3000
	Lifespan : 1576 (Years : 4 Months : 3 Days : 26)
	Write Protect: Disable
	InnoRobust: Enable
	--------------------------------------------------------------------------------------------
	ID    SMART Attributes                            Value           Raw Value
	--------------------------------------------------------------------------------------------
	[09]  Power On Hours                              [18304]         [090200646480470000000000]
	[0C]  Power Cycle Count                           [  130]         [0C0200646482000000000000]
	[AA]  Total Bad Block Count                       [   15]         [AA0300646400000F00000000]
	[AD]  Erase Count Max.                            [  883]         [AD020064642D037303000000]
	[AD]  Erase Count Avg.                            [  813]         [AD020064642D037303000000]
	[C2]  Temperature                                 [    0]         [000000000000000000000000]
	[EB]  Later Bad Block                             [    0]         [EB0200640000000000000000]
	[EB]  Read Block                                  [    0]         [EB0200640000000000000000]
	[EB]  Write Block                                 [    0]         [EB0200640000000000000000]
	[EB]  Erase Block                                 [    0]         [EB0200640000000000000000]
	[EC]  Unstable Power Count                        [    0]         [EC0200646400000000000000]
	admin@sonic-switch: ~$

## Implementation
### Generic part
#### 'show' utility update
New item under menu `platform` in `show/main.py`  
It will execute "ssdhealth -d /dev/sdX" [options]

#### ssdhealth utility
New utility in `sonic-utilities/scripts/`  
It will import device plugin `ssdutil.py` and print the output returned by different API functions  

**Syntax:**

	root@mts-sonic-dut:/home/admin# ssdutil -h
	usage: ssdhealth -d DEVICE [-h] [-v] [-e]
	
	Show disk device health status
	
	optional arguments:
	  -h, --help            show this help message and exit
	  -d, --device          disk device to get information for
	  -v, --verbose         show verbose output (more parameters)
	  -e, --vEndor          show vendor specific disk information
	
	Examples:
	  ssdutil -d /dev/sda
	  ssdutil -d /dev/sda -v
	  ssdutil -d /dev/sda -e


#### Plugins design
##### Class SsdBase
Location: `sonic-buildimage/src/sonic-platform-common/sonic_platform_base/sonic_ssd/ssd_base.py`  
Generic implementation of the API. Will use specific utilities for known disks or the `smartctl` utility for others. Since not all disk models are in smartctl's database, some information can be unavailable or incomplete.

    class SsdBase:
      ...

##### Class SsdUtil
Inherited from SsdBase. Can be implemented by vendors to provide detailed info about the disk installed.  
Location: `sonic-buildimage/device/{{vendor}}/platform/plugins/ssd_util.py`  
                           
    class SsdUtil(SsdBase):
      ...

#### API
* **get\_disk\_health(diskdev)**
	* Accepts:
		* diskdev:string  - disk device name (e.g. /dev/sda)
	* Returns:
		* res:float - Floating point in range 0-100 representing disk health in percentages. -1 if not available
* **get\_temperature(diskdev)**
	* Accepts:
		* diskdev:string  - disk device name (e.g. /dev/sda)
	* Returns:
		* res:string - Integer (floating point?) disk temperature in centigrade. Zero if not available
* **get\_model(diskdev)**
	* Accepts:
		* diskdev:string  - disk device name (e.g. /dev/sda)
	* Returns:
		* res:string - Human readable string holding disk model. Empty if not available
* **get\_firmware(diskdev)**
	* Accepts:
		* diskdev:string  - disk device name (e.g. /dev/sda)
	* Returns:
		* res:string - Human readable string holding disk firmware version. Empty if not available
* **get\_serial(diskdev)**
	* Accepts:
		* diskdev:string  - disk device name (e.g. /dev/sda)
	* Returns:
		* res:string - Human readable string holding disk serial number. Empty if not available
* **get\_vendor_output(diskdev)**
	* Accepts:
		* diskdev:string  - disk device name (e.g. /dev/sda)
	* Returns:
		* res:string - Human readable string. Output of vendor application. Empty if not available

## Utilities and packages
#### smartctl
Part of smartmontools package (1.9M)  
PR: [https://github.com/Azure/sonic-buildimage/pull/2703](https://github.com/Azure/sonic-buildimage/pull/2703)

#### iSmart
Utility for InnoDisk Corp. SSDs (<120K)
https://www.innodisk.com/en/iService/utility/iSMART  
Need to be added as binary.

#### SmartCmd
Utility for StorFly and Virtium (2.2M)  

## (Optional) Daemon for monitoring
Daemon in Pmon (ssdmond) which will periodically query disk health (get_health()) and raise alarm when value decides  to some critical value.  

## Open questions
1. Daemon and monitoring?
2. SNMP needed?
