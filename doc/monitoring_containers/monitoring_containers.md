# Monitoring and Auto-Mitigating Unhealthy Containers in SONiC

# High Level Design Document
#### Rev 0.1

# Table of Contents
* [List of Tables](#list-of-tables)
* [Revision](#revision)
* [Scope](#scope)
* [Defintions/Abbreviation](#definitionsabbreviation)
* [1 Feature Overview](#1-feature-overview)
    - [1.1 Monitoring](#11-monitoring)
    - [1.2 Auto-mitigating](#12-auto-mitigating)
    - [1.3 Requirements](#13-requirements)
        - [1.3.1 Functional Requirements](#131-functional-requirements)
        - [1.3.2 Configuration and Management Requirements](#132-configuration-and-management-requirements)
        - [1.3.3 Fast-Reboot/Warm-Reboot requirements](#133-fast-rebootwarm-reboot-requirements)
    - [1.4 Design](#14-design)
        - [1.4.1 Basic Approach](#141-basic-approach)
* [2 Functionality](#2-functionality)
    - [2.1 Target Deployment Use Cases](#21-target-deployment-use-cases)
    - [2.2 Functional Description](#22-functional-description)
        - [2.2.1 Monitoring Critical Processes](#221-monitoring-critical-processes)
        - [2.2.2 Monitoring Critical Resource Usage](#222-monitoring-critical-resource-usage)
        - [2.2.3 Auto-restart Docker Container](#223-auto-restart-docker-container)
        - [2.2.4 CLI (and usage example)](#224-cli-and-usage-example)
            - [2.2.4.1 Show the Status of Auto-restart](#2241-show-the-status-of-auto-restart)
            - [2.2.4.2 Configure the Status of Auto-restart](#2242-configure-the-status-of-auto-restart)
        - [2.2.5 CONTAINER_FEATURE Table](#225-container_feature-table)

# List of Tables
* [Table 1: Abbreviations](#definitionsabbreviation)

# Revision
| Rev |    Date    |          Author        |     Change Description    |
|:---:|:----------:|:----------------------:|---------------------------|
| 0.1 | 02/18/2020 | Yong Zhao, Joe Leveque |       Initial version     |

# Scope
This document describes the high level design of features to monitor and auto-mitigate
the unhealthy containers in SONiC.

# Definitions/Abbreviation
| Abbreviation |         Description          |
|--------------|------------------------------|
| Config DB    | SONiC Configuration Database |
| CLI          | Command Line Interface       |

# 1 Feature Overview
SONiC is a collection of various switch applications which are held in docker containers
such as BGP container and SNMP container. Each application usually includes several processes which are 
working together to provide and receive the services from other modules. As such, the health of
critical processes in each docker container is imperative not only for the docker
container working correctly but also for the intended functionalities of entire SONiC switch.

## 1.1 Monitoring
This feature is used to monitor the running status of critical processes and critical resource 
usage such as CPU, memory and disk of each docker container.

We used Monit system tool to detect whether a critical process is running or not and whether 
the resource usage of a docker container is beyond the pre-defined threshold.

## 1.2 Auto-Mitigating
This feature demonstrated docker container can be automatically shut down and
restarted if one of critical processes running in docker container exits unexpectedly. Restarting
the entire docker container ensures that configuration is reloaded and all processes in 
docker container get restarted, thus increasing the likelihood of entering a healthy state.

We leveraged the 'event listener' mechanism in supervisord to auto-restart a docker container
if one of its critical processes exited unexpectedly. We also added a configuration option to make this 
auto-restart feature dynamically configurable. Specifically users can run CLI to configure this 
feature residing in Config_DB as enabled/disabled status.

## 1.3 Requirements

### 1.3.1 Functional Requirements
1. Monit must provide the ability to generate an alert when a critical process has not
    been alive for 5 minutes.
2. Monit must provide the ability to generate an alert when the resource usage of
    a docker container is larger than the pre-defined threshold.
3. The event listener in supervisord must receive the signal when a critical process in 
    a docker container crashed or exited unexpectedly and then restart this docker 
    container.
4. CONFIG_DB can be configured to enable/disable this auto-restart feature for each docker
    container.. 
5. Users can access the status of auto-restart feature via the CLI utility
    1. Users can see current auto-restart status for docker containers.
    2. Users can configure auto-restart status for a specific docker container.

### 1.3.2 Configuration and Management Requirements
Via the init_cfg.json file, these container features are disabled by default.
Configuration of these features can be done via:
1. config_db.json
2. CLI

### 1.3.3 Fast-Reboot/Warm-Reboot Requirements
During the fast-reboot/warm-reboot/warm-restart procedures in SONiC, a select number of processes
and the containers they reside in are stopped in a special manner (via a signals or similar).
In this situation, we need ensure these containers remain stopped until the fast-reboot/warm-reboot/warm-restart
procedure is complete. Therefore, in order to prevent the auto-restart mechanism from restarting 
the containers prematurely, it is the responsibility of the fast-reboot/warm-reboot/warm-restart 
procedure to explicitly stop the systemd service which manages the container immediately after stopping
and critical processes/container. Once the systemd service is explicitly stopped, it will not attempt
to automatically restart the container.


## 1.4 Design

### 1.4.1 Basic Approach
Monitoring the running status of critical processes and resource usage of docker containers
depends on the Monit system tool. Since Monit natively provides a mechanism
to check whether a process is running or not, it will be straightforward to integrate this into monitoring 
the critical processes in SONiC. However, Monit only provides a method to monitor the resource
usage on a per-process level not a per-container level. As such, monitoring the resource usage of a docker 
container is not as straightforward. In our design, we propose to utilize the mechanism with
which Monit can spawn a process and check the return value of the process. We will have Monit
launch a script which reads the resource usage of the container and compares the resource usage
with a configured threshold value for that container. If the current resource usage is less than
the configured threshold value, the script will return 0 and Monit will not log a message.
However, if the resource usage exceeds the threshold, the script will return a non-zero value
and Monit will log an alert message to the syslog.

We employed the 'event listener' mechanism in supervisord to achieve auto-restarting docker 
containers. We configure our event listener to listen for process exit events. When a supervised
process exits, supervisord will pass the event to our custom event listener. The event listener
determines if the process is a critical process  and whether it exited unexpectedly. If both of
these conditions are true, the event listener will kill the supervisord process. Since supervisord
runs as PID 1 inside the containers, when supervisord exits, the container will stop. When the
container stops, the systemd service which manages the container will also stop, but it is
configured to automatically restart the service, thus it will restart the container.

# 2 Functionality
## 2.1 Target Deployment Use Cases
These two features are used to perform the following functions:
1. Monit will write an alert message into syslog if one if critical process has not been
    alive for 5 minutes.
2. Monit will write an alert message into syslog if the usage of memory is larger than the
    pre-defined threshold for a docker container.
3. A docker container will auto-restart if one of its critical processes crashed or exited
    unexpectedly.

## 2.2 Functional Description


### 2.2.1 Monitoring Critical Processes
Monit natively implements a mechanism to monitor whether a process is running or not. In detail,
Monit will periodically read the target processes from configuration file and try to match 
those process with the processes tree in Linux kernel.

Below is an example of Monit configuration file to monitor the critical processes in lldp
container.

*/etc/monit/conf.d/monit_lldp*
```bash
###############################################################################
# Monit configuration file for lldp container
# Process list:
#   lldpd
#   lldp_syncd
#   lldpmgrd
###############################################################################
check process lldp_monitor matching "lldpd: "
    if does not exit for 5 times within 5 cycles then alert
check process lldp_syncd matching "python2 -m lldp_syncd"
    if does not exit for 5 times within 5 cycles then alert
check process lldpmgrd matching "python /usr/bin/lldpmgrd"
    if does not exit for 5 times within 5 cycles then alert
```
However, Monit is unable to monitor multiple processes executing the same command but with
different arguments. For example, in teamd container, there are multiple teamd processes 
running the same command ```/usr/bin/teamd``` but using different port channel as argument.
Since there exists 1:1 mapping between a port channel and a teamd process, we employ Monit to 
monitor a script which retrieves all the port channels from Config_DB and then determine
whether there exists a teamd process in Linux for each port channel. If succeed, that means
all teamd processes are live. Otherwise, we will know at least teamd process exited unexpectedly
and then Monit will write an alert message into syslog. Similarly we can also use this method
to solve the same issue in dhcp_relay container.

### 2.2.2 Monitoring Critical Resource Usage
Similar to monitoring the critical processes, we can employ Monit to monitor the resource usage
such as CPU, memory and disk for each process. Unfortunately Monit is unable to do the resource monitoring
in the container level. Thus we propose a new design to achieve such monitoring based on Monit.
Specifically Monit will monitor a script and check its exit status. This script
will correspondingly read the resource usage of docker containers, compare it with
pre-defined threshold and then return a value. The value 0 signified that
the resource usage is less than threshold and non-zero means Monit will send an alert since
current usage is larger than threshold.

Below is an example of Monit configuration file for lldp container to pass the pre-defined 
threshold (bytes) to the script and check the exiting value.

```bash
check program container_memory_lldp with path "/usr/bin/memory_checker lldp 104857600"
    if status != 0 then alert
```

We will employ similar mechanism for CPU and disk utilization. Thresholds for each resource,
per container can be determined by the operator by examining averages of resource usage in
a production environment. The value `0` in table represents the corresponding feature in 
the docker container is in `disabled` status.


### 2.2.3 Auto-restart Docker Container
The design principle behind this auto-restart feature is docker containers can be automatically shut down and
restarted if one of critical processes running in the container exits unexpectedly. Restarting
the entire container ensures that configuration is reloaded and all processes in the container
get restarted, thus increasing the likelihood of entering a healthy state.

Currently SONiC used supervisord system tool to manage the processes in each
docker container. Actually auto-restarting docker container is based on the process 
monitoring/notification framework. Specifically
if the state of process changes for example from running to exited,
an event notification `PROCESS_STATE_EXITED` will be emitted by supervisord.
This event will be received by event listener. The event listener determines if the process is
critical process and whether it exited unexpectedly. If both of
these conditions are true, the event listener will kill the supervisord process. Since supervisord
runs as PID 1 inside the containers, when supervisord exits, the container will stop. When the
container stops, the systemd service which manages the container will also stop, but it is
configured to automatically restart the service, thus it will restart the container.

We also introduced a configuration option which can enable or disable this auto-restart feature
dynamically according to the requirement of users. In detail, we created a table 
named `CONTAINER_FEATURE` in Config_DB and this table includes the status of
auto-restart feature for each docker container. Users can easily use CLI to
check and configure the corresponding docker container status.

### 2.2.4 CLI (and usage example)
The CLI tool will provide the following functionality:
1. Show current status of auto-restart feature for docker containers.
2. Configure the status of a specific docker container.

#### 2.2.4.1 Show the Status of Auto-restart
```
admin@sonic:~$ show container feature autorestart
Container Name         Status 
--------------------  --------
database               disabled
lldp                   disabled
radv                   disabled
pmon                   disabled
sflow                  enabled
snmp                   enabled
telemetry              enabled
bgp                    disabled
dhcp_relay             disabled
rest-api               enabled
teamd                  disabled
syncd                  enabled
swss                   disabled
```

#### 2.2.4.2 Configure the Status of Auto-restart
```
admin@sonic:~$ sudo config container feature autorestart database enabled
```

### 2.2.5 CONTAINER_FEATURE Table
Example:
```
{
    "CONTAINER_FEATURE": {
        "database": {
            "auto_restart": "enabled",
            "high_mem_alert": "157286400",
            "high_cpu_alert": "0",
            "high_disk_alert": "0"
        },
        "lldp": {
            "auto_restart": "disabled",
            "high_mem_alert": "104857600",
            "high_cpu_alert": "0",
            "high_disk_alert": "0"
        },
        "radv": {
            "auto_restart": "disabled",
            "high_mem_alert": "104857600",
            "high_cpu_alert": "0",
            "high_disk_alert": "0"
        },
        "pmon": {
            "auto_restart": "disabled",
            "high_mem_alert": "104857600",
            "high_cpu_alert": "0",
            "high_disk_alert": "0"
        },
        "sflow": {
            "auto_restart": "enabled",
            "high_mem_alert": "0",
            "high_cpu_alert": "0",
            "high_disk_alert": "0"
        },
        "snmp": {
            "auto_restart": "enabled",
            "high_mem_alert": "157286400",
            "high_cpu_alert": "0",
            "high_disk_alert": "0"
        },
        "telemetry": {
            "auto_restart": "enabled",
            "high_mem_alert": "0",
            "high_cpu_alert": "0",
            "high_disk_alert": "0"
        },
        "bgp": {
            "auto_restart": "disabled",
            "high_mem_alert": "314572800",
            "high_cpu_alert": "0",
            "high_disk_alert": "0"
        },
        "dhcp_relay": {
            "auto_restart": "disabled",
            "high_mem_alert": "104857600",
            "high_cpu_alert": "0",
            "high_disk_alert": "0"
        },
        "rest-api": {
            "auto_restart": "enabled",
            "high_mem_alert": "0",
            "high_cpu_alert": "0",
            "high_disk_alert": "0"
        },
        "teamd": {
            "auto_restart": "disabled",
            "high_mem_alert": "104857600",
            "high_cpu_alert": "0",
            "high_disk_alert": "0"
        },
        "syncd": {
            "auto_restart": "enabled",
            "high_mem_alert": "629145600",
            "high_cpu_alert": "0",
            "high_disk_alert": "0"
        },
        "swss": {
            "auto_restart": "disabled",
            "high_mem_alert": "157286400",
            "high_cpu_alert": "0",
            "high_disk_alert": "0"
        },
    }
}
```
