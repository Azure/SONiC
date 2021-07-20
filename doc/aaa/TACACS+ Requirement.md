# SONiC TACACS+ Protocol Requirement

# Table of Contents
- [Table of Contents](#table-of-contents)
- [About this Manual](#about-this-manual)
- [1 Functional Requirements](#1-functional-requirement)
  * [1.1 Authentication](#11-authentication)
  * [1.2 Authorization](#12-authorization)
  * [1.3 Accounting](#13-accounting)
  * [1.4 User script](#14-user-script)
  * [1.5 Docker support](#15-docker-support)
  * [1.6 Multiple TACACS server](#16-multiple-tacas-server)
- [2 Configuration and Management Requirements](#2-configuration-and-management-requirements)
  * [2.1 SONiC CLI](#21-sonic-cli)
  * [2.2 Config DB](#22-config-db)
  * [2.3 Counter](#23-counter)
  * [2.4 System log](#24-system-log)
- [3 Limitation](#limitation)
  * [3.1 Command size](#31-command-size)
  * [3.2 Server count](#32-server-count)

# About this Manual
This document provides a detailed description on the requirement of TACACS+ protocol support.

# 1 Functional Requirement
## 1.1 Authentication
- Authentication when user login to SONiC host.


## 1.2 Authorization
- Authorization when:
	- User login to SONiC host.
	- User run any executable file or script on SONiC host.
		- The full path and parameters will be send to TACACS+ server side for authorization.
		- For recursive command/script, only the top level command have authorization.
		- No authorization for  bash builtin command and bash function, but if a bash function call any executable file or script, those executable file or script will have authorization.

- Supported Authorization types:
	- EXEC: user session authorization support. this happen when user login.
	- Command: user run command in shell.

- Failover:
	- Authorization will happen before execute, if remote TACACS server not available, use local group based authorization as failover.


## 1.3 Accounting
 - Accounting is the action of recording what a user is doing, and/or has done.
 
 - Following event will be accounted:
 	- User login to SONiC host.
 	- User logout.
 	- User run command on host: 
		- Command start run.
		- Command finish.
		- For recursive command/script, only the top level command have Accounting.

- Failover:
	- Use syslog as backup when remote TACACS not not accessible from SONiC.


## 1.4 User script
 - User can create and run their own script. 
 - If user create a script but TACACS+ service side not have configuration to allow user run this script, user script will be blocked by authorization.

## 1.5 Docker support
 - Docker exec command will be covered by Authorization and Accounting. 
 - SONiC AAA can't cover any command user run inside a docker.

## 1.6 Multiple TACACS server
 - Support config multiple TACACS server.
 - When a server not available, will try next server as backup.
 - When all server not accessible from SONiC, use native failover solution.

# 2 Configuration and Management Requirements
## 2.1 SONiC CLI
 - Enable/Disable TACACS Authorization/Accounting command
```
    config tacacs authorization enable
    config tacacs authorization disable
    config tacacs accounting enable
    config tacacs accounting disable
```

 - Whitelist management command
```
    show tacacs whitelist
    config tacacs whitelist remove <command>
    config tacacs whitelist add <command>
```

 - Counter command
```
    show tacacs counter
    sonic-clear tacacscounters
```

## 2.2 Config DB
 - TACACS AAA are fully configable by config DB.

## 2.3 Counter
 - Support  AAA counter:
```
    show tacacs counter
    
        server1: 10.1.1.45
        Messages sent: 24
        Messages received: 20
        Requests accepted: 14
        Requests rejected: 8
        Requests timeout: 2
        Requests retransmitted: 1
        Bad responses: 1
```

## 2.4 Syslog
- Generate syslog when Authentication/Authorization/Accounting.
- When remote TACACS server not accessible from SONiC, use syslog for accounting.


# 3 Limitation
## 3.1 Command size
 - TACACS protocol limittation: command + parameter size should smaller than 240 byte. The longer than 240 bytes parts will be drop.
 	- This limitation is a protocol level, all TACACS implementation have this limittation, include CISCO, ARISTA and Cumulus.
 	- Both Authorization and Accounting have this limitation.
 	- When user user a command longer than 240 bytes, only commands within 240 bytes will send to TACACS server. which means Accounting may lost some user input. and Authorization check can only partly check user input.


## 3.2 Server count
 - Max TACACS server  count was hardcoded, default count is 8.

