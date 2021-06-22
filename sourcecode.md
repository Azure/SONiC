# SONiC Source Repositories


## Imaging and Building tools

### sonic-buildimage  
- https://github.com/Azure/sonic-buildimage
    - Main repo that contains SONiC code,links to all sub-repos, build related files, platform/device specific files, etc.,
	This repo has the following directories.
	- device - It contains files specific to each vendor device. In general, it contains the python scripts for accessing EEPROM, SFP, PSU, LED,etc., specific to the device hardware.
	- dockers - This folder contains sub-folders for all dockers running in the SONiC. Each of those sub-folders contains files that explains about the processes that need to run inside that docker. List of dockers and the processes running inside each docker is given at the end of this document.
	- files - Contain multiple sub-folders required for building and running SONiC services.   
		(a) Aboot,
		(b) apt - few default files related to  for "apt-get" ,"apt-*" applications 
		(c) build_templates - Contains the jinja2 template files to generate (as part of "build process") the systemd services files required for starting the various dockers. It also contains the file sonic_debian_extension.j2 is used in "build process"; it copies the required files and installs the required packages in the "/fsroot" which is built ass part of the SONiC image.  
		(d) dhcp - Contains the config file for dhcp client & exit hook scripts, (e) docker - Contains the "docker" file (related to docker-engine) that is extracted from docker-ce 17.03.0\~ce-0\~debian-stretch to enable 'service docker start' in the build chroot env.  
		(f) image_config - Contains sub-folders like apt (non debian packages related info), bash (for bashrc), caclmgrd (control plane ACL manager daemon), cron.d (logrotate), ebtables (filter), environment (vtysh,banner), fstrim, hostcfgd (tacacs+ config handler), hostname (service to handle hostname config), interfaces (service to handle interface related config changes), logrotate, ntp (ntp service with conf file j2 file), platform (rc.local file), rsyslog (service for syslog & j2 file),snmp (snmp.yml file), sudoers (sudo users & permissions),  systemd (journald.conf file), updategraph (script for getting minigraph and installing it), warmboot-finalizer (script used during warmreboot).  
		(g) initramfs-tools - Contains files related to ramfs, (h) scripts - contains scripts for arp_update (gratuitous ARP/ND), swss, syncd, etc., (i) sshd - SSH service and keygen script  
		
	- installer - contains scripts that are used by onie-mk-demo script that is called as part of build_image.sh
	- rules - contains the "config" file where the build options can be modified, contains *.mk makefiles that contains the required marcros for building the image.
	- platform - contains sub-folders for all platforms like "barefoot", "broadcom", "cavium", "centec", "marvell", "mellanox", "nephos", "p4", "vs" (virtual switch).
	  Each of those platform folder contains code specific to the hardware device from each vendors. It includes the required kernel drivers, platform sensors script for fetching data from hardware devices, etc.,
	- sonic-slave, sonic-slave-stretch - Contains the main Dockerfile that lists the various debian packages that are required for various features.
	- src - contains sub-folders for features like bash, gobgp, hiredis, initramfs-tools, iproute2, isc-dscp, ixgbe, libnl3, libteam, 
	  libyang, lldpd, lm-sensors, mpdecimal, python-click, python3, radvd - Router advertisement for IPv6, redis, smartmontools, 
	  snmpd, socat, sonic-config-engine, sonic-daemon-base, sonic-device-data, sonic-frr (routing software with patches), supervisor,
	  swig, tacacs, telemetry and thrift.


## SAI, Switch State Service

### sonic-swss  
- https://github.com/Azure/sonic-swss
	- Switch State Service - Core component of SONiC which processes network switch data - The SWitch State Service (SWSS) is a collection of software that provides a database interface for communication with and state representation of network applications and network switch hardware.

	- This repository contains the source code for the swss container, teamd container & bgp container shown in the [architecture diagram](https://github.com/Azure/SONiC/blob/master/images/sonic_user_guide_images/section4_images/section4_pic1_high_level.png "High Level Component Interactions")
	- When swss container is started, start.sh starts the processes like rsyslogd, orchagent, restore_neighbors, portsyncd, neighsyncd, swssconfig, vrfmgrd, vlanmgrd, intfmgrd, portmgrd, buffermgrd, enable_counters, nbrmgrd, vxlanmgrd & arp_update.

  SWWS repository contains the source code for the following.
  - cfgmgr - This directory contains the code to build the following processes that run inside swss container. More details about each deamon is available in the [architecture document](https://github.com/Azure/SONiC/wiki/Architecture).
	- nbrmgrd - manager for neighbor management - Listens to neighbor-related changes in NEIGH_TABLE in ConfigDB for static ARP/ND configuration and also to trigger proactive ARP (for potential VxLan Server IP address by not specifying MAC) and then uses netlink to program the neighbors in linux kernel. nbrmgrd does not write anything in APP_DB.
	- portmgrd - manager for Port management - Listens to port-related changes in ConfigDB and sets the MTU and/or AdminState in kernel using "ip" commands and also pushes the same to APP_DB.
	- buffermgrd - manager for buffer management - Reads buffer profile config file and programs it in ConfigDB and then listens (at runtime) for cable length change and speed change in ConfigDB, and sets the same into buffer profile table ConfigDB.
	- teammgrd - team/portchannel management - Listens to portchannel related config changes in ConfigDB and  runs the teamd process for each port channel. Note that teammgrd will be executed inside teamd container (not inside swss container).
	- intfmgrd - manager for interfaces - Listens for IP address changes and VRF name changes for the interfaces in ConfigDB and programs the same in linux using "/sbin/ip" command and writes into APP_DB.
    - vlanmgrd - manager for VLAN - Listens for VLAN related changes in ConfigDB and programs the same in linux using "bridge" & "ip" commands and and writes into APP_DB
    - vrfmgrd - manager for VRF - Listens for VRF changes in ConfigDB and programs the same in linux and writes into APP_DB.
	
  - fpmsyncd - this folder contains the code to build the "fpmsynd" process that runs in bgp container. This process runs a TCP server and listens for messages from Zebra for route changes (in the form of netlink messages) and it writes the routes to APP_DB. It also waits for clients to connect to it and then provides the route updates to those clients.
  - neighsyncd - this folder contains the code to build the "neighsyncd" process. Listens for ARP/ND specific netlink messages from kernel for dynamically learnt ARP/ND and programs the same into APP_DB.
  - portsyncd - this folder contains the code to build the "portsyncd" process. It first reads port list from configDB/ConfigFile and adds them to APP_DB. Once if the port init process is completed, this process receives netlink messages from linux and it programs the same in STATE_DB (state OK means port creation is successful in linux).
  - swssconfig - this folder creates two executables, viz, swssconfig and swssplayer. 
     - "swssconfig" runs during boot time only. It restores FDB and ARP table during fast reboot. It takes the port config, copp config, IP in IP (tunnel) config and switch (switch table) config from the ConfigDB and loads them into APP_DB.
	 - "swssplayer" - this records all the programming that happens via the SWSS which can be played back to simulate the sequence of events for debugging or simulating an issue.
  - teamsyncd - allows the interaction between “teamd” and south-bound subsystems. It listens for messages from teamd software and writes the output into APP_DB.
  - orchagent - The most critical component in the Swss subsystem. Orchagent contains logic to extract all the relevant state injected by *syncd daemons, process and massage this information accordingly, and finally push it towards its south-bound interface. This south-bound interface is yet again another database within the redis-engine (ASIC_DB), so as we can see, Orchagent operates both as a consumer (for example for state coming from APPL_DB), and also as a producer (for state being pushed into ASIC_DB).

	
### sonic-swss-common  	
- https://github.com/Azure/sonic-swss-common  
	- Switch State Service common library - Common library for Switch State Service

### Opencomputeproject/SAI  
- https://github.com/opencomputeproject/SAI (Switch Abstraction Interface standard headers)
	- This repo refers/uses the SAI sub-repo from OCP github that includes the required SAI header files.

### sonic-sairedis  
- https://github.com/Azure/sonic-sairedis
	- This repo contains the C++ library code for interfacing to SAI objects in Redis
	- The SAI Redis provides a SAI redis service that built on top of redis database. 
	- It contains two major components 
	   - a SAI library that puts SAI objects into the ASIC_DB and
	   - a syncd process that takes the SAI objects and puts them into the ASIC.
	- It also contains the sub-folders "saiplayer" (that records all the actions from orchagent that results in making the SAI API calls to ASIC), "saidump" ( tool to dump the ASIC contents)
	- Note that the SAI library for the specific platform is not part of this repo. The SAI library is built using the sonic-buildimage/platform/<platformname>/*sai.mk (slave.mk includes the platform/<platformname>/rules.mk that in turn includes the *sai.mk that installs the required SAI debians).
	   

### sonic-dbsyncd  
- https://github.com/Azure/sonic-dbsyncd
	- Python Redis common functions for LLDP
	- This repo contains the code for SONiC Switch State Service sync daemon for LLDP data. Scripts upload lldp information to Redis DB


### sonic-py-swsssdk  
- https://github.com/Azure/sonic-py-swsssdk 
  - This repo contains python utility library for SWSS DB access. 
  - configdb.py - This provides utilities like ConfigDBConnector, db_connect, connect, subscribe, listen, set_entry, mod_entry, get_entry, get_keys, get_table, delete_table, mod_config, get_config, etc.,
  - dbconnector.py - It contains utilities like SonicV1Connector, SonicV2Connector, etc.,
  - exceptions.py - It contains utilities like SwssQueryError, UnavailableDataError, MissingClientError, etc.,
  - interface.py - It contains utilities like DBRegistry, DBInterface, connect, close, get_redis-client, publish, expire, exists,  keys, get, get_all, set, delete, etc.,
  - port_util.py - It contains utilities like get_index, get_interface_oid_map, get_vlan_id_from_bvid, get_bridge_port_map, etc.,
  - util.py - It contains utilities like process_options, setup_logging, etc.,


### sonic-quagga  
- https://github.com/Azure/sonic-quagga/tree/debian/0.99.24.1  
  - This repo contains code for the Quagga routing software which is a free software that manages various IPv4 and IPv6 routing protocols. Currently Quagga supports BGP4, BGP4+, OSPFv2, OSPFv3, RIPv1, RIPv2, and RIPng as well as very early support for IS-IS.

	
## Monitoring and management tools  

### sonic-mgmt  
- https://github.com/Azure/sonic-mgmt
	- Management and automation code used for build, test and deployment automation

### sonic-utilities   
- https://github.com/Azure/sonic-utilities  
  - This repository contains the code for Command Line Interfaces for SONiC. 
  - Folders like "config", "show", "clear" contain the CLI commands 
  - Folders like "scripts", "sfputil", "psuutil" & "acl_loader" contain the scripts that are used by the CLI commands. These scripts are not supposed to be directly called by user. All these scripts are wrapped under the "config" and "show" commands.
  - "connect" folder and "consutil" folder is used for scripts to connec to other SONiC devices and manage them from this device.
  - crm folder contains the scripts for CRM configuration and show commands. These commands are not wrapped under "config" and "show" commands. i.e. users can use the "crm" commands directly.
  - pfc folder contains script for configuring and showing the PFC parameters for the interface
  - pfcwd folder contains the PFC watch dog related configuration and show commands.
  - utilities-command folder contains the scripts that are internally used by other scripts.


### sonic-snmpagent  
- https://github.com/Azure/sonic-snmpagent
  - This repo contains the net-snmpd AgentX SNMP subagent implementation for supporting the MIBs like MIB-II, Physical Table MIB, Interfaces MIB, Sensor Table MIB, ipCidrRouteDest table in IP Forwarding Table MIB, dot1qTpFdbPort in Q-BRIDGE-MIB & LLDP MIB.
  - The python scripts present in this repo are used as part of the "snmp" docker that runs in SONiC.


## Switch hardware drivers

### sonic-linux-kernel  
- https://github.com/Azure/sonic-linux-kernel
- This repo contains the Kernel patches for various device drivers. 
- This downloads the appropriate debian kernel code, applies the patches and builds the custom kernel for SONiC.


### sonic-platform-common  
- https://github.com/Azure/sonic-platform-common
  - This repo contains code which is to be shared among all platforms for interfacing with platform-specific peripheral hardware.
  - It contains the APIs for implementing platform-specific functionality in SONiC
  - It provides the base class for peripherals like EEPROM, LED, PSU, SFP, chassis, device, fan, module, platform, watchdog, etc., that are used for existing platform code as well as for the new platform API.
  - Platform specific code present in sonic-buildimage repo (device folder) uses the classes defined in this sonic-platform-common repository.
  - New platform2.0 APIs are defined in the base classes inside "sonic_platform_base" folder. 

### sonic-platform-daemons  
- https://github.com/Azure/sonic-platform-daemons
  - This repo contains the daemons for controlling platform-specific functionality in SONiC
  - This repo contains python scripts for platform daemons that listens for events from Optics, LED & PSU and writes them in the STATE_DB
  - xcvrd - This listens for SFP events and writes the status to STATE_DB.
  - ledd - This listens for LED events and writes the status to STATE_DB.
  - psud - This listens for PSU events and writes the status to STATE_DB.


### Other Switch Hardware Drivers (Deprecated)    
- https://github.com/celestica-Inc/sonic-platform-modules-cel
- https://github.com/edge-core/sonic-platform-modules-accton
- https://github.com/Azure/sonic-platform-modules-s6000
- https://github.com/Azure/sonic-platform-modules-dell
- https://github.com/aristanetworks/sonic
- https://github.com/Ingrasys-sonic/sonic-platform-modules-ingrasys


## Dockers Information  

Following are the dockers that are running in SONiC. 

1) telemetry - Runs processes like telemetry & dialout_client_cli
2) syncd - Runs processes like syncd & dsserve which is used to sync the application data into the ASIC.
3) dhcp_relay - Runs the DHCP relay agent process.
4) teamd - Runs the teammgrd and teamsyncd processes.
5) radv (router-advertise) - Runs the IPv6 router advertisement process
6) snmp - Runs the SNMP agent daemon
7) swss (orchagent) - Runs the orchagent, portsyncd, neighsyncd, vrfmgrd, vlanmgrd, intfmgrd, portmgrd, buffermgrd, nbrmgrd & vxlanmgrd.
8) pmon (platform-monitor) - Runs the platform daemons xvrd (listens for SFP events) & psud (listens for power supply related events).
9) lldp - Runs the lldp process and lldpmgrd
10) bgp (fpm-frr) - Runs bgpcfgd, zebra, staticd, bgpd & fpmsyncd
11) database - Runs the REDIS server.
