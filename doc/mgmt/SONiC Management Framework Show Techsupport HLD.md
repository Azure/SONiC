# Show Techsupport
Diagnostic information aggregated presentation
# High Level Design Document
#### Rev 0.1
# Table of Contents

  - [List of Tables](#list-of-tables)
  - [Revision](#revision)
  - [About this Manual](#about-this-manual)
  - [Scope](#scope)
  - [Definition/Abbreviation](#definition-abbreviation)
  - [1 Feature Overview](#1-feature-overview)
    * [1.1 Requirements](#11-requirements)
      + [1.1.1 Functional Requirements](#111-functional-requirements)
      + [1.1.2 Configuration and Management Requirements](#112-configuration-and-management-requirements)
      + [1.1.3 Scalability Requirements](#113-scalability-requirements)
      + [1.1.4 Warm Boot Requirements](#114-warm-boot-requirements)
    * [1.2 Design Overview](#12-design-overview)
      + [1.2.1 Basic Approach](#121-basic-approach)
      + [1.2.2 Container](#122-container)
      + [1.2.3 SAI Overview](#123-sai-overview)
  - [2 Functionality](#2-functionality)
    * [2.1 Target Deployment Use Cases](#21-target-deployment-use-cases)
    * [2.2 Functional Description](#22-functional-description)    
    * [2.3 Functionality Caveats](#23-functionality-caveats)
      + [2.3.1 Effect on execution of other Commands](#231-effect-on-execution-of-other-commands)
  - [3 Design](#3-design)
    * [3.1 Overview](#31-overview)
    * [3.2 DB Changes](#32-db-changes)
      + [3.2.1 CONFIG DB](#321-config-db)
      + [3.2.2 APP DB](#322-app-db)
      + [3.2.3 STATE DB](#323-state-db)
      + [3.2.4 ASIC DB](#324-asic-db)
      + [3.2.5 COUNTER DB](#325-counter-db)
    * [3.3 Switch State Service Design](#33-switch-state-service-design)
      + [3.3.1 Orchestration Agent](#331-orchestration-agent)
      + [3.3.2 Other Process](#332-other-process)
    * [3.4 SyncD](#34-syncd)
    * [3.5 SAI](#35-sai)
    * [3.6 User Interface](#36-user-interface)
      + [3.6.1 Data Models](#361-data-models)
      + [3.6.2 CLI](#362-cli)
        - [3.6.2.1 Configuration Commands](#3621-configuration-commands)
        - [3.6.2.2 Show Commands](#3622-show-commands)
        - [3.6.2.3 Debug Commands](#3623-debug-commands)
      + [3.6.3 REST API Support](#363-rest-api-support)
  - [4 Flow Diagrams](#4-flow-diagrams)
    * [4.1 Show Techsupport Process Flow](#41-show-techsupport-process-flow)
  - [5 Error Handling](#5-error-handling)
  - [6 Serviceability and Debug](#6-serviceability-and-debug)
  - [7 Warm Boot Support](#7-warm-boot-support)
  - [8 Scalability](#8-scalability)
  - [9 Unit Test](#9-unit-test)
  - [10 Internal Design Information](#10-internal-design-information)
    * [10.1 Overview](#101-overview)
    * [10.2 Management Framework Context](#102-management-framework-context)
    * [10.3 Host Context](#103-host-context)

  <small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

# List of Tables

## Revision
| Rev |     Date    |       Author       | Change Description                |
|:---:|:-----------:|:------------------:|-----------------------------------|
| 0.1 | 10/06/2019  |   Kerry Meyer      | Initial version                   |
| 0.2 | 04/02/2021  |   Kerry Meyer      | Revised for submission in SONiC community PR https://github.com/Azure/SONiC/pull/756                                      |


# About this Manual
This manual describes the user interface for obtaining aggregated diagnostic information for the SONiC subsystem via the Management Framework infrastructure.
# Scope
The scope of the information contained in this document is the high level design for the "show techsupport" command implementation under the control of the Management Framework infrastructure. It is intended to cover the general approach and method for providing a flexible collection of diagnostic information items. It also considers the basic mechanisms to be used for obtaining the various types of information to be aggregated. It does not address specific details for collection of all supported classes of information.

# Definition/Abbreviation

# 1 Feature Overview

Provide Management Framework functionality to process the "show techsupport" command:

    - Create an aggregated file containing the information items needed for
      analysis and diagnosis of problems occurring during switch operations.
    - Support reduction of aggregated log file information via an
      optional "--since" parameter specifying the desired logging start time.

NOTE: The underlying feature for which this Management Framework feature provides "front end" client interfaces is unchanged by the addition of these interfaces. (The "since <date>" option available through these interfaces, however, is restricted to the IETF/YANG date/time format.) Please refer to the following document for a description of the "show techsupport" base feature:

https://github.com/Azure/sonic-utilities/blob/master/doc/Command-Reference.md#troubleshooting-commands    

## 1.1 Requirements

### 1.1.1 Functional Requirements

Provide a Management Framework based interface for the "show tech-support" command.

### 1.1.2 Configuration and Management Requirements
Provide the ability to invoke the command via the following client interfaces:

     - Management Framework CLI (same syntax as the existing Click-based
        API except for tighter restriction of the "DateTime" format to
        conform with the Yang/IETF DateTime standard)
     - REST API
     - gNOI

(See Section 3 for additional details.)

### 1.1.3 Scalability Requirements

Time and storage space constraints: The large number of information items collected and the potentially large size of some of the items (e.g. interface information display in a large system) present an exposure to the risk of long processing times and significant demands on disk storage space. The Management Framework interface invokes the same command used for the Click-based interface. It adds no significant additional overhead or processing time. The storage space requirements are unchanged.
### 1.1.4 Warm Boot Requirements
N/A
## 1.2 Design Overview
### 1.2.1 Basic Approach
This feature will be implemented using the Management Framework infrastructure supplemented with customized access mechanisms for handling "non-DB" data items.

### 1.2.2 Container
The user interface (front end) portion of this feature is implemented within the Management Framework container.

### 1.2.3 SAI Overview
N/A (non-hardware feature)

# 2 Functionality
## 2.1 Target Deployment Use Cases
This feature provides a quick and simple mechanism for network administrators or other personnel with no detailed knowledge of switch internal details to gather an extensive set of information items by using a single command. These items provide critical information to help development and sustaining engineering teams in the analysis and debugging of problems encountered in deployment and test environments.
## 2.2 Functional Description
The set of items to be gathered for a given software release is defined by the development team. It is specified in a way that enables run-time access to the desired set of information items to be collected. The definition of the set of information items to be collected includes specification of the access function to be used for each item in the list. Each access function gathers a subset of the required information, formats it as needed, and packs it into the output file. The location of the resulting output file is provided to the requesting client at the completion of command execution.

The output file name has the following form:

`
/var/dump/sonic_dump_sonic_YYYYMMDD_HHMMSS.tar.gz
`

Example:

`/var/dump/sonic_dump_sonic_20191118_221625.tar.gz
`
See section 3.6.2.2 for an explanation of the output file name format.

To view the contents of the file, the user must copy it to a local file in the client file system. If the file is to be extracted within the directory to which it is copied, the directory should have at least 50 MB of available space. To extract the file inside of the directory to which it has been copied while displaying a list of output files, the following command can be used:

`
tar xvzf filename.tar.gz
`

The files are extracted to a directory tree, organized based on the type of information contained in the files. Example file categories for which sub-directories are provided in the output file tree include:

- log files ("log" directory )
- Linux configuration files ("etc" directory)
- generic application "dump" output ("dump" directory)
- network hardware driver information ("sai" directory)
- detailed information on various processes ("proc" directory).

To extract the file contents to an alternate location, the following form of the "tar" command can be used:

`
 tar xvzf filename.tar.gz -C /path/to/destination/directory
`

Some of the larger "extracted" files are compressed in gzip format. This includes log files and core files and also includes other files containing a large amount of output (e.g. a dump of all BGP tables). These files have a ".gz" file type. They can be extracted using:

`
gunzip <filename.gz>
`
## 2.3 Functionality Caveats

The current implementation of the "show techsupport" command has the following limitations. The user should be aware of these limitations when using the command.

### 2.3.1 Effect on execution of other Commands

During execution of the "show techsupport" command, execution of many of the other Management Framework commands is delayed. It is possible to issue and initate all other Management Framework commands in parallel with "show techsupport" command execution via other Management Framework sessions. However, completion of execution for commands requiring inter-process or external docker communication is delayed until after completion of "show techsupport" command execution. This is a result of serialization of these commands via a global resource lock. This limitation is not specific to the "show techsupport" command, but is a generic limitation of the current Management Framework implementation.

# 3 Design
## 3.1 Overview
The "show techsupport" command causes invocation of an RPC sent from the management framework to a process in the host to cause collection of a list of flexibly defined sets of diagnostic information (information "items"). The collected list of items is stored in a compressed "tar" file with a unique name. The command output provides the location of the resulting compressed tar file.

The "since" option can be used, if desired, to restrict the time scope for log files and core files to be collected. This option is passed to the host process for use during invocation of the applicable information gathering sub-functions.

## 3.2 DB Changes
N/A
### 3.2.1 CONFIG DB
### 3.2.2 APP DB
### 3.2.3 STATE DB
### 3.2.4 ASIC DB
### 3.2.5 COUNTER DB

## 3.3 Switch State Service Design
N/A
### 3.3.1 Orchestration Agent
### 3.3.2 Other Process
The "show techsupport" feature requires RPC support in a process running within the host context. The host process handling the RPC is the SONiC Host Services D-Bus server. It is responsible for dispatching "show techsupport" requests from the management framework container to a SONiC Host Services D-Bus “servlet” for the “show techsupport” command to trigger allocation of an output file, gathering and packing of the required information into the output file, and sending of a response to the management framework RPC agent to specify the name and path of the output file.

## 3.4 SyncD
N/A

## 3.5 SAI
N/A

## 3.6 User Interface
### 3.6.1 Data Models
The following Sonic Yang model is used for implementation of this feature:

```
module: sonic-show-techsupport

rpcs:
  +---x sonic-show-techsupport-info
     +---w input
     |  +---w date?   yang:date-and-time
     +--ro output
        +--ro output-status?     string
        +--ro output-filename?   string

```



### 3.6.2 CLI
#### 3.6.2.1 Configuration Commands
N/A
#### 3.6.2.2 Show Commands

Command syntax summary:

`
show techsupport [since <DateTime>]
`

Command Description:

Gather information for troubleshooting. Display the name of a file containing the resulting group of collected information items in a compressed "tar" file.


Syntax Description:

|    Keyword    | Description |
|:--------------|:----------- |
| since \<DateTime\> | This option uses a text string containing the desired starting Date/Time for collected log files and core files. The format of the Date/Time in the string is defined by the Yang/IETF date-and-time specification (REF http://www.netconfcentral.org/modules/ietf-yang-types, based on http://www.ietf.org/rfc/rfc6020.txt). If "since \<DateTime\>" is specified, this value  is passed to the host process for use during invocation of the applicable log/core file gathering sub-functions.|

Command Mode: User EXEC

Output format example and summary:

```
Example:

Output stored in:  /var/dump/sonic_dump_sonic_20191008_082312.tar.gz

--------------------------------------------------

Output file name sub-fields are defined a follows:

- YYYY = Year
- MM = Month (numeric)
- DD = Day of the Month
- HH = hour of the current time (based on execution of the Linux "date" command) at the start of command execution
- MM = minute of the current time (based on execution of the Linux "date" command) at the start of command execution
- SS = second of the current time (based on execution of the Linux "date" command) at the start of command execution
```

Command execution example (basic command):

```
sonic# show techsupport

Output stored in:  /var/dump/sonic_dump_sonic_20191008_082312.tar.gz

```
Command execution Example (using the "since" keyword/subcommand):

```
sonic# show tech-support
  since  Collect logs and core files since a specified date/time
  |      Pipe through a command
  <cr>   

sonic# show tech-support since
  String  date/time in the format:

 "YYYY-MM-DDTHH:MM:SS[.ddd...]Z" or
 "YYYY-MM-DDTHH:MM:SS[.ddd...]+hh:mm" or
 "YYYY-MM-DDTHH:MM:SS[.ddd...]-hh:mm" Where:

 YYYY = year, MM = month, DD = day,
 T (required before time),
 HH = hours, MM = minutes, SS = seconds,
 .ddd... = decimal fraction of a second (e.g. ".323")
 Z indicates zero offset from local time
 +/- hh:mm indicates hour:minute offset from local time

sonic# show tech-support since 2019-11-27T22:02:00Z
Output stored in:  /var/dump/sonic_dump_sonic_20191127_220334.tar.gz
```
Command execution example invocation via REST API:

```
REST request via CURL:

curl -X POST "https://10.11.68.13/restconf/operations/sonic-show-techsupport:sonic-show-techsupport-info" -H  "accept: application/yang-data+json" -H  "Content-Type: application/yang-data+json" -d "{  \"sonic-show-techsupport:input\": {    \"date\": \"2019-11-27T22:02:00.314+03:08\"  }}"

Request URL:

https://10.11.68.13/restconf/operations/sonic-show-techsupport:sonic-show-techsupport-info

Response Body:

{
  "sonic-show-techsupport:output": {
    "output-status": "Success",
    "output-filename": "/var/dump/sonic_dump_sonic_20191128_013141.tar.gz",
  }
}
```

Command execution example invocation via gNOI API:

```
root@sonic:/usr/sbin# ./gnoi_client -module Sonic -rpc showtechsupport -jsonin "{\"input\":{\"date\":\"2019-11-27T22:02:00Z\"}}" -insecure
Sonic ShowTechsupport
{"sonic-show-techsupport:output":{"output-status": "Success","output-filename":"/var/dump/sonic_dump_sonic_20191202_194856.tar.gz"}}
```

NOTE: See section 3.6.1 for a description of the limitations of the current implementation. A supplementary capability to transfer the tech support file and other diagnostic information files to the client via the Management Framework interface is highly desirable for a future release.

#### 3.6.2.3 Debug Commands
N/A

### 3.6.3 REST API Support
REST API support is provided. The REST API corresponds to the SONiC Yang model described in section 3.6.1.

# 4 Flow Diagrams
## 4.1 Show Techsupport Process Flow
![ShowTechsupport process flow](images/showtech-flow-diagram.JPG)

# 5 Error Handling
in the event of a command timeout or a crash during execution due to a resource shortage, the user can retry after the failure/reboot that occurred during the first execution. Also, in some cases, the tar file will be available, despite the failure, in the host /var/dump directory.

# 6 Serviceability and Debug
Any errors encountered during execution of the "show tech-support" command that prevent retrieval or saving of information are reported in the command output at completion of the operation.

# 7 Warm Boot Support
N/A

# 8 Scalability
Refer to section 1.1.3

# 9 Unit Test

|    Case    | Trigger | Result |
|:-----------|:--------|:-------|
| Basic command execution | Execute the "show techsupport" command with no parameters. | Confirm that the command is accepted without errors and a "result" file name is returned. Confirm that the result file contains the expected set of items. (Examine/expand the contents of the file to ensure that the top level directory tree is correct and that the number of sub-files within the tar file is correct.)|
"since" option (postive test case) | Execute the command with the "--since" TEXT option with a valid date string specifying a time near the end of one of the  unfiltered output items from the first test.| Same as the "Basic command execution" case. Additionally, confirm that the expected time filtering has occurred by examining one of the affected sub-files.|
"since" option (negative test case #1)|Execute the command with the "--since" TEXT option with an invalid date string.|Verify that an error is returned.|
"since" option (negative test case #2)|Execute the command with the "--since" TEXT option with no date string.|Verify that an error is returned.|Execute the command with the "--since" option with no date string.| Verify that an error is returned.|





# 10 Internal Design Information
Please refer to the diagram in Section 4.1, referenced below:

[4.1 Show Techsupport Process Flow](#41-show-techsupport-process-flow)

## 10.1 Overview
The Management Framework container (a Docker container) uses the SONiC D-Bus RPC mechanism specified in "[SONiC Docker to Host communication](https://github.com/mikelazar/SONiC/blob/69bb868dec98fc05b8b046f0925ca2e89604c49a/doc/mgmt/Docker%20to%20Host%20communication.md)" to trigger execution of the "generate_dump" Bash script on the SONiC host and to receive a response providing the result.

## 10.2 Management Framework Context
 Execution in the SONiC Management Framework docker of the "show tech-support" CLI command or the equivalent REST/gNOI invocation causes the corresponding "actioner" script to be run from the context of the Management Framework docker. This script invokes the REST API generated from the "show tech-support" Yang definition. The corresponding API handler function, registered as a SONiC D-Bus client, initiates an asynchronous D-Bus host query and relays the response, containing the location of a "techsupport bundle" file if execution is successful, back to the Management Framework interface (CLI, REST, or gNOI) from which the request was received. (In the event of an error, it instead returns the error message received from the "show techsupport" servlet running within the context of the server process for the SONiC D-Bus host services object.)

## 10.3 Host Context
Within the SONiC host context, execution of the "show techsupport" command is initiated when the SONiC D-Bus host facility dispatches a request received from the Management Framework docker by invoking a script (servlet) registered with the SONiC D-Bus host server for handling of the "show techsupport" command. This servlet invokes the "generate_dump" Bash script, spawning a process that collects a "bundle" of items providing diagnostic information for processes running on the switch, packs the collected information into a compressed .tar file, and returns the location of the resulting file to the "show techsupport" D-Bus servlet script on successful completion. (In the event of an error, it instead returns an error message describing the error.) The servlet, via the SONiC host services D-Bus server, then sends the resulting RPC response back to the "show techsupport" client in the SONiC Management Framework docker via the SONiC D-Bus RPC infrastructure.
