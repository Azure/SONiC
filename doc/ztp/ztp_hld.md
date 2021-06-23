# Zero Touch Provisioning (ZTP)
# High Level Design

### Rev 1.0

# Table of Contents
  * [Revision](#revision)
  * [Requirements](#requirements)
  * [DHCP Options](#dhcp-options)
  * [Flow](#flow)
  * [Error Cases](#error-cases)
  * [ZTP CLI](#ztp-cli)
  * [Action Items](#action-items)
  * [Phases](#phases)

# Revision
| Rev | Date     |  Author       | Change Description                |
|:---:|:------------:|:------------------:|-----------------------------------|
| v0.1 |   11/03/18   |    Simone Salman   | Initial version                   |
| v0.2 |   11/14/18   |    Simone Salman   | Addressing Review comments        |

# Requirements
- Build option to have ZTP enabled
- SNMP should be enabled during the ZTP process (achieved through leveraging updategraph)
- ZTP should verify, download and install SONiC image
- Updategraph handles all configuration during ZTP service
- ZTP should receive post config validation script, that collects switch info and sends it back to a remote location for processing
- ZTP status should be logged through syslog
- Interruption of ZTP service should set ZTP status as incomplete/interrupted and ZTP should be disabled.

# DHCP Options
The following are the private DHCP options used during the implementation of updategraph/ZTP

| DHCP Option | Information         | Explanation                                       |
|:-----------:|:-------------------|:-------------------------------------------------|
|    224      | snmp_community     | usage implemented through updategraph |
|    225      | minigraph_url     | usage implemented through updategraph |
|    226      | acl_url           | usage implemented through updategraph |
|    227      | ztp_image_url     | URL for the SONiC image for the switch to dowload |
|    228      | validation_url    | URL for the validation script -- a script that runs post config and  collects device info and sends it for processing to a remote location designated within the validation script |

# Flow
![](https://github.com/simone-dell/SONiC/blob/ztp/images/ztp_hld/ztp_flow.jpg)

For ZTP using DHCP, provisioning initially takes place over the management network and is initiated through a DHCP hook. A DHCP option is used to specify a configuration script. This script is then requested from the Web server and executed locally on the switch.
1.	Simplify installation of switch, the steps involved will be
    - Rack and Stack 
    - Connect 
    - Power-on 
2.	Onie Boots into SONiC (with ZTP enabled) from flash
    - Power-on ZTP service will take over to load the initial configuration.
    - ZTP service flag $enabled is true
    - ZTP service flag $post_install is false
    - ZTP service flag $post_config is false
3.	The switch now enters ZTP mode and does the following,
    - Obtains an IP address from the DHCP server
    - Obtains the URL for the SONIC image, minigraph, and post config validation script 
4.	At this step, the switch will have information to reach the HTTP / TFTP server through URLs given by DHCP server.
5.	ZTP service will enable and start updategraph service
    - Within updategraph, ztp_enabled is true, and post_install is false
    - This triggers updategraph to configure the device with an default config
    - Upon exiting updategraph, SNMP will start
6.	ZTP service will now download the software image file.
7.	sonic_installer will validate image to be a SONIC image compatible with the current platform.
8.	sonic_installer  will check remaining hard disk for space before doing image install.
9.	ZTP service will install image using sonic-installer
    - ZTP service flag $post_install is true
10.	ZTP service will enable updategraph
11.	ZTP service will call for reboot to apply new SONIC image, with option ZTP-enabled –post_install
12.	Upon reboot, if ZTP service flag $post_install is true, ZTP hands off to updategraph
13.	The configuration script is received and applied by updategraph, returns to ZTP service
ZTP service flag $post_config is true
14.	Validation:
    - SONIC device will download post config validation script through ZTP enabled DHCP option
    - The script is customized based on user preference
    - The script collects local info and sends it back to server using information in the script itself
15.	Delete old SONIC image upon successful reimage/ ZTP status check:
    - Check if $post_install is true
    - Check if $post_config is true
    - Set $enable to false
    - Verify no errors were thrown during the service
    - Output to syslog server and var/log/syslog

### Updategraph
Needs to be updated to provide default config to the switch when ztp_enabled is true
Needs to continue as currently designed when ztp_enabled is true and post_install is true
Can be used to download config file and apply config

### Interruption of ZTP
Through command line utility “ZTP Disable” 
Sets ZTP $enabled flag to FALSE
If ZTP is interrupted mid-service, output is shown
If ZTP service receives “NA_ZTP” flag, continue updategraph with default config
ZTP service needs to have separate option for interruption than updategraph
If ZTP service is interrupted, updategraph should still be enabled and allowed to continue

### Syslog
ZTP status should be logged to syslog server and /var/log/syslog

### Build Option
Need to allow build time option to compile with ZTP enabled

# Error Cases
- Case: Switch receives invalid SONiC image URL from DHCP server
    - Switch should output error logs to syslog and apply default config.  Kill the ZTP service with aborted status, and allow other services to come up
- Case: Switch receives invalid configuration URL from DHCP server
    - Switch should output error logs to syslog and apply default config.  Kill the ZTP service with aborted status, and allow other services to come up
- Case: Switch receives invalid post-config validation URL from DHCP server
    - Switch should output error logs to syslog.  Disable the ZTP service, and allow other services to come up
- Case: Switch receives invalid OS image 
    - Switch should output error logs to syslog and apply default config.  Kill the ZTP service, and allow other services to come up
- Case: Switch does not have enough memory to store the image
    - Switch should output error logs to syslog and apply default config.  Kill the ZTP service, and allow other services to come up
- Case: Switch receives invalid configuration from DHCP server
    - Switch should output error logs to syslog, and gracefully exit the ZTP service, logging that image install completed successfully, and configuration script is invalid.
- Case: Post-validation script fails
    - The success on the script is processed by a remote server.  ZTP service is exited gracefully, logging with image install completed successfully.
 

# ZTP CLI
The ZTP CLI will allow the user to manually start, stop, and obtain status of the ZTP service.

Will show the user whether ZTD is enabled, the date of the last execution of the ZTP script, and the completion status of the ZTP service

    show ztp-status
The ZTP status will show the user the current SONiC image, if the configuration was successful, and if the post-configuration script was run, as well as the time of the last successful completion of the ZTP service

    config ztp enable
The user can re-enable ZTP after a failed state using the CLI.  This puts the switch into “factory setting” and reloads with ZTP enabled, allowing for provisioning restart.

    config ztp disable
The user can interrupt the ZTP service:
- If cancelled before image download, ZTP service will exit
- If cancelled after image download but before reboot, ZTP service will output sonic_installer list and exit
- If cancelled after reboot, ZTP service will not apply any configuration, and exit 
- If cancelled after configuration, ZTP service will not continue with the validation script, and exit


# Action Items

## ZTP SERVICE
- The service needs to start immediately after boot
- If ZTP is not enabled, exit the service
- If ZTP is enabled and post_install is false, run updategraph to apply default configuration
- Acquire image and configuration script from HTTP/TFTP server
- Install the image using sonic-installer
- Reload to reimage the device
- ZTP is enabled and post_install is true, apply configurations using updategraph
- Run post_validation script
- Exit ZTP 
- Test to see if Ansible server is reachable for further configuration
## Updategraph
- If ZTP is enabled and post_install is false, apply default configs to the switch and exit
- If ZTP is enabled and post_install is true, continue to apply configs based on graph_url

# Check-in Phases
## Phase 1
- Implement image download and install
- Implement image and config validation
## Phase 2
- Post install script: downloaded after updategraph finishes
- Command line utility
## Phase 3
- ZTP interrupt process
- ZTP test plan

