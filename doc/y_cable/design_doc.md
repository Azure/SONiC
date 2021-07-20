## Multiple Y-Cable vendor API design

## Revision

| Rev | Date     | Author          | Change Description |
|:---:|:--------:|:---------------:|--------------------|
| 0.1 | 03/22/21 | Vaibhav Dahiya  | Initial version    |

## Scope

### Overview

This document summarizes the approach taken to accommodate multiple
Y-Cable vendors for supporting dual-ToR configurations in SONiC.
All the design choices are presented
such that vendors can implement the Y-Cable interface with ease and effeciency
and all the feature requirements are met.


Challenge: to provide an interface for Y-Cable to interact with PMON docker and HOST
           which can be uniform across all vendors. As part of this refactor, we will
           be moving towards a model where only xcvrd interacts with the cables/transceivers, 
           and host-side processes will communicate with xcvrd rather than with the devices directly


### Vendor to Module mapping

#### Background

- We need to have a vendor to corresponding Y-Cable module mappings defined and accessible somewhere for PMon container to import

  - Before calling any Y-Cable API, how do we know which type/vendor does the Y-Cable belong to?
  - Also lets suppose once the type/vendor of the cable is known from where does interested application import the module?
  - Since there can be many specs which vendors follow (8636, 8624 etc.) for transceivers it is important to have a solution which meets all requirements so that the API access is available to whoever wants to invoke it.
  - another issue is how do we support the multiple vendors modules/packages which could be used by both SONiC PMon container docker as well as the SONiC CLI (sonic-utilities). Basically the package should be available both in host and container. Although we are moving towards a model where CLI should not need to import Y-Cable modules but it is still preferred have it accessible

#### Proposed Solution

- We define a file which would contain a mapping from vendor/part number to the appropriate Y-Cable module to load and put it in a place which is accessible to both PMon container and host which can be present in sonic_platform_common
- It makes sense to keep the mapping file in the sonic_y_cable package so that it can be updated in the same pull request when a vendor adds a new cable implementation. However, we cannot install data files outside of the Python directory via a wheel. Considering this we propose to make this a Python file which simply contains a 2D dictionary which can be used to look up the module path. If the file is part of the sonic_y_cable package, it will be installed in both the PMon container and the host

- For example
    ```
        /sonic_platform_common/sonic_y_cable/y_cable_vendor_mapping.py
    ```

Vendors can have several implementations/ways to use this concept
- Vendor has a 1:1 mapping between model and Python file
- Vendor has one Python file which supports all models
- A combination of the above

- Mapping could be such that its a dictionary in 2D

    ```python
    {
         "<vendor_name_1>": {
             "<model_name>": "<module>"
         },
         "<vendor_name_2>" : {
             "<model_name>": "<module>"
         }
    }
    ```

- For example

    ```python
    {
         "Vendor1": {
             "model1": "vendor1/y_cable.py"
         },

         "Vendor2" : {
             "model_a": "vendor2/y_cable_a.py",
             "model_b": "vendor2/y_cable_b.py"
         }
    }
    ```

#### Rationale

  - This is because both the container and host can easily access this path. And it would be easy to mount this on container (PMon container) as applicable.
  - The sonic_y_cable package is installed in both host and pmon

#### Implementation details
- Once the xcvrd will start (daemon launched) or transceiver is connected (plugged in), in both cases xcvrd can determine the Vendor name and part number from the register specification and then it can appropriately read and parse the vendor name and part number from the spec using eeprom. Once it has this information it can then use this file to Load/import the appropriate module.

### Directory Structure

#### Background

- We need to have an appropriate place where vendors can implement their modules and provide SONiC with an abstraction layer where PMon container and host can access the Y-Cable packages of all the vendors with ease

  - Currently the assumption is Y-Cable API will be written in the Python language
  - If a vendor has an existing library in a different language, they will need to either find a way to wrap/bind it in Python to align with the description below or (preferrably) provide a pure Python implementation
  - Since sonic_y_cable (sonic-platform-common/sonic_y_cable) is already built as a package today (only for a single Y-Cable vendor with sonic-buildimage) vendors can also place their implementation in this directory itself.
  - Also, a vendor can provide multiple files to support multiple cables/groups of cables. Importing the module from the mapping should import all the necessary implementation for the cable
  - Vendors can put their common implementation across multiple modules in helper files which can be present in the vendor directory or inside another subdirectory. The second example below shows this approach.

#### Proposed Solution

- Vendors can place their implementations in this format

    ```
    sonic_platform_common/sonic_y_cable/<vendor>/<module>
    ```

    ```
    sonic_platform_common/sonic_y_cable/<vendor>/<module_1>
    sonic_platform_common/sonic_y_cable/<vendor>/<module_2>
    ```
- Few examples

    ```
    sonic_platform_common/sonic_y_cable/vendor1/y_cable.py
    ```

    ```
    sonic_platform_common/sonic_y_cable/vendor2/y_cable_xyz.py
    sonic_platform_common/sonic_y_cable/vendor2/y_cable_abc.py
    sonic_platform_common/sonic_y_cable/vendor2/abc_helper.py
    ```
#### Rationale
- The requirement here would be that the vendors must create their modules such that It can easily be accessed/imported and the modules also adhere to a uniform convention

  - Vendor name should be derivable from the vendor name field inside the register spec of transceiver type.
  - This is essential so that host/PMon container can determine the vendor name from eeprom and import the module of the transceiver appropriately


### Port to module mapping
- Another thing that is required is once we do have a mapping from vendor/part number to the appropriate Y-Cable module to load, we need to map appropriate port to a module as well

#### Background

  - Basically, the key idea is once we have a port identified as being of a certain vendor and it has been identified to load a certain module which we can gather from the mapping described above we still need to call the correct module on each port each time we call the API on the port
  - want to maintain this mapping in memory since xcvrd does not want to read/parse this y_cable_vendor_mapping.py file again and again each time we call the Y-Cable API
  - Also note that the module loaded might change during xcvrd running lifetime since cable can be changed from one vendor to another. So we need to take this into consideration as well

#### Proposed Solution

  - Each module of the Y-Cable vendor can be a class (of each transceiver type) and all we need to do is instantiate the objects of these classes as class instances and these objects will provide the interface of calling the API's for the appropriate vendor Y-Cable.
  - This instantiation will be done inside xcvrd, when xcvrd starts
  - These objects basically emulate Y-Cable instances and whatever action/interaction needs to be done with the Y-Cable the methods of these objects would provide that
  - each vendor in their implementation can inherit from a base class where there will be definitions for all the supported capabilities of the Y-Cable.
  - for vendors the recommended approach in case their subclass implementation does not support a method, is to set the method equal to None. This differentiates it from a method they forgot to implement. Then, the calling code should first check if the method is None before attempting to call it.

  - For example the base class would be like this

    ```python
        class YCableBase(object):
            def __init__(self, port):
                self.port = port
                <function body here>

            def toggle_mux_to_torA(self, port):
                <function body here>

            def toggle_mux_to_torB(self, port):
                <function body here>

            def check_prbs(self, port):
                <function body here>

            def enable_loopback(self, port):
                <function body here>

            def get_eye_info(self, port):
                <function body here>

    ```

  - For example a typical module of the vendor can be like this

    ```python
        class YCable(YCableBase):

            def __init__(self):
                <function body here>

            def toggle_mux_to_torA(self, port):
                <function body here>

            def toggle_mux_to_torB(self, port):
                <function body here>

            def check_prbs(self, port):
                <function body here>

            def enable_loopback(self, port):
                <function body here>

            def get_eye_info(self, port):
                <function body here>

    ```
#### Implementation details
- Now for xcvrd to use this solution, all it has to do is instantiate the class objects defined by importing the appropriate module. This should happen when xcvrd starts or if there is a change event  (xcvrd inserted/removed)

    ```python
        # import the vendor to module mapping
        from sonic_y_cable import y_cable_vendor_mapping
        from sonic_py_common.general import load_module_from_source

        # This vendor/part_name determination subroutines will either read through eeprom
        # or else read from the TRANSCEIVER_INFO_TABLE inside state db

        vendor_name = get_vendor_name(port)

        part_name = get_part_name(port)

        path_to_module_obtained_from_mapping = y_cable_vendor_mapping.mapping[vendor_name][part_name]

        module = load_module_from_source("y_cable",  "path_to_module_obtained_from_mapping")

        from module import YCable

        y_cable_port_Ethernet0 = YCable(port)

        # and to use the Y-Cable APIS
        try:
            y_cable_port_Etherne0.toggle_mux_to_torA()
        except:
            helper_logger.log(not able to toggle_mux_to_torA)

    ```
#### Rationale

  - xcvrd can maintain a dict of these objects an whenever it needs to call the API for the correct port (in this case physical port) it can easily achieve so by indexing into the instance of the physical port and calling the class object.

    ```python
        Y_cable_ports_instances = {}

        # appending instances in the dictionary
        Y_cable_ports_instances[physical_port] = YCable(port)

        # and to use the Y-Cable APIS
        try:
            Y_cable_ports_instances[physical_port].toggle_mux_to_torA()
        except:
            helper_logger.log(not able to toggle_mux_to_torA)
    ```


### How does SONiC CLI interact with Y-Cable API's
- Another thing that is required is once we do have a a mapping from vendor/part number to the appropriate Y-Cable module to load, we need to map appropriate port to a module as well

#### Background

  - Another requirement we have is SONiC CLI also requires to interact with the Y-Cable directly. This basically implies that all  Y-Cable vendor packages needs to be imported inside SONiC-utilities/host as well
  - This would come in the form of commands like setting PRBS, enabling disabling loopback and also get the BER info and EYE info etc
  - Also commands such as config/show hwmode is important which gives the CLI ability to toggle the mux without going into SONiC modules like mux-mgr or orchagent.
  - All these require access to Y-Cable APIs to be directly called by the CLI. But then again same problems arrive, how do we know which type/vendor does the cable/port belong to, how to load the appropriate module etc

#### Proposed Solution(s)

  - One way is since CLI lives in the host, we can choose to do everything on xcvrd lines. Meaning once there is port number, convert to a physical port and look into a mapping from vendor/part number to the appropriate Y-Cable module to load file and then load the module and execute the API
  - The more preferred approach here is CLI can interact with PMon container thorugh redis-db. Basically we can define a schema table for different operations which need to be performed on the Y-Cable. 

  - Exanple table and operations
    ```
        HW_MUX_OPERATION_TABLE|Ethernet0
        state active/standby
        enable_prbs true/false
        check_mux_direction true/false
    ```
  - In xcvrd we can have a thread running and listening to these events and execute the APIs since all the ports already have the correct mapping of the module (maintained in the form of a dictionary). The response of the API can be again written back to the redis table and absorbed by the CLI

### Concurrency for Y-Cables API

#### Background

  - Since some vendors need to execute two or three transactions in a single API it is imperative to have a solution where we can protect the Y-Cable APIs for concurrent behavior

#### Proposed Solution(s)

  - We might need to pass a semaphore or lock to the API to protect reexecuting or executing some other API in the meantime.
  - For now we insist on vendors to acquire/release locks in the API/implementation itself. So that if there is any possibility of concurrency the locking mechanism inside the the API prevents that from happening


### Storing the state within a Y-Cable Module

#### Background

  - Another requirement in future that could arise is what if we need a feature which requires vendors to store a state with the Y-Cable. In this regard it is important to address this requirement in the initial design itself. (example port to bus mapping)

#### Proposed Solution(s)

  - Since the vendor has class definitions associated with Y-Cable module, it can have attributes which store the information. However, it still needs to be decided how these attributes will be instantiated/configured. Whether this onus will be on xcvrd/daemon or whether it can be done within the module itself.

