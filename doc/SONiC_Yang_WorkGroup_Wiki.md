
# SONiC YANG MODELS AND RELATED LIBRARIES.

# Table of Contents
- [SONiC YANG MODELS AND RELATED LIBRARIES.](#sonic-yang-models-and-related-libraries)
- [Table of Contents](#table-of-contents)
- [Revision](#revision)
- [SONiC Yang models:](#sonic-yang-models)
	- [Summarized Guidelines from SONiC_YANG_Model_Guidelines.md:](#summarized-guidelines-from-sonic_yang_model_guidelinesmd)
	- [How to Write YANG Models:](#how-to-write-yang-models)
			- [Example of INTERFACE table to understand ext:key-regex-configdb-to-yang:](#example-of-interface-table-to-understand-extkey-regex-configdb-to-yang)
- [SONiC Yang models TESTs:](#sonic-yang-models-tests)
	- [Test 1: YANG Tree](#test-1-yang-tree)
		- [Generate YANG Tree, see no error in it.](#generate-yang-tree-see-no-error-in-it)
	- [Test 2: Successful data tree creation.](#test-2-successful-data-tree-creation)
	- [Test3: Check constraints such as Must, Pattern, When, Mandatory etc.](#test3-check-constraints-such-as-must-pattern-when-mandatory-etc)
	- [Test4: default statement and value in Data tree.](#test4-default-statement-and-value-in-data-tree)
- [Development and Validation of new Yang Models.](#development-and-validation-of-new-yang-models)

# Revision

| Rev |     Date    |       Author       | Change Description                |
|:---:|:-----------:|:------------------:|-----------------------------------|
| 1.0 |             | Praveen Chaudhary  | Initial version                   |
|	  |             |                    |									 |


This document will describe cover below in details. These components together will comprise of SONiC config management system (SCMS).

1.)   SONiC Yang models.

2.)   SONiC Yang models Tests.

3.)   Python Library for YANG. (PLY)

4.)   Python Library for YANG Tests.

5.)   Development and Validation of new Yang Models.

# SONiC Yang models:
SONiC YANG Models are written mainly using below 2 documents:

1.) First document describes guidelines, which are written considering the ABNF format of configDB, RFC7950 and RFC 7951.

https://github.com/Azure/SONiC/blob/master/doc/mgmt/SONiC_YANG_Model_Guidelines.md
and

2.) Second document describes the configuration for various networking objects in SONiC which are currently supported:

https://github.com/Azure/SONiC/wiki/Configuration.

## Summarized Guidelines from SONiC_YANG_Model_Guidelines.md:
-- It is mandatory to define a top level YANG container named as 'sonic-{feature}' i.e. same as YANG model name.

-- Define namespace as "http://github.com/Azure/{model-name}.

-- Each primary section of ABNF.json (i.e a dictionary in ABNF.json) for example, VLAN, VLAN_MEMBER, INTERFACE in ABNF.json will be mapped to a container in YANG model.

-- Each leaf in YANG module should have same name (including their case) as corresponding key-fields in ABNF.json.

-- Data Node/Object Hierarchy of the objects in YANG models will be same as for all the fields at same hierarchy in Config DB.

-- If an object is part of primary-key in ABNF.json, then it should be a key in YANG model.

-- If any key used in current table refers to other table, use leafref type for the key leaf definition.

-- All must, when, pattern and enumeration constraints can be derived from .h files or from code.

-- If a List object is needed in YANG model to bundle multiple entries from a Table in ABNF.json, but this LIST is not a valid entry in config data, then we must define such list as <TABLE_NAME>_LIST.


Kindly go through guideline document in detail to see the example for each constraint.

## How to Write YANG Models:
For Config DB Entry for PORT:
```
"PORT": {
    "Ethernet0": {
        "index": "0",
        "lanes": "101,102",
        "description": "fortyGigE1/1/1",
        "mtu": "9100",
        "alias": "fortyGigE1/1/1",
        "speed": "40000"
    },
    "Ethernet1": {
        "index": "1",
        "lanes": "103,104",
        "description": "fortyGigE1/1/2",
        "mtu": "9100",
        "alias": "fortyGigE1/1/2",
        "admin_status": "up",
        "speed": "40000"
    }
}
```

YANG model for PORT may be written as below:
Commented line below i.e. in "/* */" are the lines from guideline document, This guideline is being followed while writing next line in YANG model.
```
module sonic-port{

    /* To have latest feature of 1.1 */
	yang-version 1.1;
    
    /* Define namespace as "http://github.com/Azure/{model-name}. */
	namespace "http://github.com/Azure/sonic-port"; 
	prefix port;

    /* if using ietf inet type, i.e. prefix, addresses */
    /* Note, Some data type maybe processed in very 
    particular way by LibYang */
	import ietf-inet-types {
		prefix inet;
	}

    /* All sonic specific types are in below file. */
	import sonic-types {
		prefix stypes;
		revision-date 2019-07-01;
	}

    /* All sonic specific extensions are in below file. 
    Extensions are used to translation between sonicYang 
    Config(7951) and config DB config.*/
	import sonic-extension {
		prefix ext;
		revision-date 2019-07-01;
	}

	description "PORT yang Module for SONiC OS";

	revision 2019-07-01 {
		description "First Revision";
	}

    /* It is mandatory to define a top-level YANG container named 
    as 'sonic-{feature}' i.e. same as YANG model name. */
	container sonic-port {
        /* Each primary section of ABNF.json (i.e a dictionary 
        in ABNF.json) for example, VLAN, VLAN_MEMBER, INTERFACE 
        in ABNF.json will be mapped to a container in YANG model. */
		container PORT {

			description "PORT part of config_db.json";

            /* If a List object is needed in YANG model to bundle 
            multiple entries from a Table in ABNF.json, but this 
            LIST is not a valid entry in config data, then we must
            define such list as <TABLE_NAME>_LIST */
			list PORT_LIST {

                /* If an object is part of primary-key in ABNF.json
                , then it should be a key in YANG model. */
				key "port_name";

                /* Below extension helps in conversion from configDB 
                to sonicYang.json. Primary Keys under PORT table will 
                be matched with this key.*/
				ext:key-regex-configdb-to-yang "^(Ethernet[0-9]+)$";
                
                /* Below extension helps in conversion from 
                sonicYang.json to configDB.json, i.e. port_name 
                field will be used as primary key */
				ext:key-regex-yang-to-configdb "<port_name>";

                /* Each leaf in YANG module should have same 
                name (including their case) as corresponding 
                key-fields in ABNF.json.*/
				leaf port_name {
					type string {
						length 1..128;
					}
				}

				leaf alias {
					type string {
						length 1..128;
					}
				}

				leaf lanes {
					mandatory true;
					type string {
						length 1..128;
					}
				}

				leaf description {
					type string {
						length 0..255;
					}
				}

				leaf speed {
					mandatory true;
					type uint32 {
						range 1..100000;
					}
				}

				leaf mtu {
					type uint16 {
						range 1..9216;
					}
				}

				leaf index {
					type uint16 {
						range 0..256;
					}
				}

				leaf admin_status {
					type stypes:admin_status;
				}

				leaf fec {
					type string {
					    /* All must, when, pattern and enumeration 
					    constraints can be derived from .h files or 
					    from code. */
						pattern "rs|fc|None";
					}
				}
			} /* end of list PORT_LIST */

		} /* end of container PORT */

	} /* end of container sonic-port */

} /* end of module sonic-port */
```

How sonic_yang.json will look like:
Note: RFC 7951 describes how the config for yang models will be in json format, In this document sonic_yang.json describes json config as per RFC 7951.
```
"sonic-vlan:sonic-vlan": {
			"sonic-vlan:VLAN_MEMBER": {
				"VLAN_MEMBER_LIST": [{
					"vlan_name": "Vlan100",
					"port": "Ethernet156",
					"tagging_mode": "tagged"
				}]
			},
			"sonic-vlan:VLAN": {
				"VLAN_LIST": [{
					"vlan_name": "Vlan100",
					"description": "server_vlan"
				}]
			}
		},
		"sonic-port:sonic-port": {
			"sonic-port:PORT": {
				"PORT_LIST": [{
						"port_name": "Ethernet0",
						"alias": "eth0",
						"description": "Ethernet0",
						"speed": 25000,
						"mtu": 9000,
						"admin_status": "up"
					},
					{
						"port_name": "Ethernet1",
						"alias": "eth1",
						"description": "Ethernet1",
						"speed": 25000,
						"mtu": 9000,
						"admin_status": "up"
					}
				]
			}
		}
```

#### Example of INTERFACE table to understand ext:key-regex-configdb-to-yang:
```
"INTERFACE": {
			"Ethernet112": {},
			"Ethernet14": {},
			"Ethernet16": {},
			"Ethernet18": {},
			"Ethernet112|2a04:5555:40:a709::2/126": {
				"scope": "global",
				"family": "IPv6"
			},
			"Ethernet112|10.184.228.211/31": {
				"scope": "global",
				"family": "IPv4"
			}
}
```
As shown above, interface table has 2 types of entries. So yang models will have 2 separate _LIST(s) as shown in yang model below. Notice that ext:key-regex-configdb-to-yang for each list is written in a way that it matches with only one _LIST. 

For example regex "^(Ethernet[0-9]+)|([a-fA-F0-9:./]+)$" will match with "Ethernet112|2a04:5555:40:a709::2/126" and with "Ethernet112|10.184.228.211/31" in above interface config. Also '()' in regex are used to fetch matched groups. For regex: "^(Ethernet[0-9]+)|([a-fA-F0-9:./]+)$" and for key "Ethernet112|2a04:5555:40:a709::2/126" the matched group will give "Ethernet112" and "2a04:5555:40:a709::2/126". These matched group will be assigned in keys specified for the list, i.e. key "port_name ip-prefix";

YANG Models for INTERFACE:
```
container sonic-interface {

		container INTERFACE {

			description "INTERFACE part of config_db.json";

			list INTERFACE_LIST {

				description "INTERFACE part of config_db.json with vrf";

				key "port_name";

				ext:key-regex-configdb-to-yang "^(Ethernet[0-9]+)$";

				ext:key-regex-yang-to-configdb "<port_name>";

			    leaf port_name {
					type leafref {
						path /port:sonic-port/port:PORT/port:PORT_LIST/port:port_name;
					}
				}

				leaf vrf_name {
					type string {
						pattern "Vrf[a-zA-Z0-9_-]+";
						length 3..255;
					}
				}
			}
			/* end of INTERFACE_LIST */

			list INTERFACE_IPPREFIX_LIST {

				description "INTERFACE part of config_db.json with ip-prefix";

				key "port_name ip-prefix";

				ext:key-regex-configdb-to-yang "^(Ethernet[0-9]+)|([a-fA-F0-9:./]+)$";

				ext:key-regex-yang-to-configdb "<port_name>|<ip-prefix>";

				leaf port_name {
					/* This node must be present in INTERFACE_LIST */
					must "(current() = ../../INTERFACE_LIST[port_name=current()]/port_name)"
					{
						error-message "Must condition not satisfied, Try adding PORT: {}, Example: 'Ethernet0': {}";
					}

					type leafref {
						path /port:sonic-port/port:PORT/port:PORT_LIST/port:port_name;
					}
				}

				leaf ip-prefix {
					type union {
						type stypes:sonic-ip4-prefix;
						type stypes:sonic-ip6-prefix;
					}
				}

				leaf scope {
					type enumeration {
						enum global;
						enum local;
					}
				}

				leaf family {

					/* family leaf needed for backward compatibility
					   Both ip4 and ip6 address are string in IETF RFC 6021,
					   so must statement can check based on : or ., family
					   should be IPv4 or IPv6 according.
					 */

					must "(contains(../ip-prefix, ':') and current()='IPv6') or
						(contains(../ip-prefix, '.') and current()='IPv4')";
					type stypes:ip-family;
				}
			}
			/* end of INTERFACE_IPPREFIX_LIST */

		}
		/* end of INTERFACE container */
	}
}
```

sonic_yang.json:
RFC 7951
```
		"sonic-interface:sonic-interface": {
			"sonic-interface:INTERFACE": {
				"INTERFACE_LIST": [{
					"port_name": "Ethernet8"
				}],
				"INTERFACE_IPPREFIX_LIST": [{
					"port_name": "Ethernet8",
					"ip-prefix": ""2a04:xxxx:xx:7777::1/64",
					"scope": "global",
					"family": "IPv6"
				}]
			}
		},
```

# SONiC Yang models TESTs:

Yang tests are mainly written in below two files:

Json file contains the config to test in sonic_yang format.

https://github.com/Azure/sonic-buildimage/blob/master/src/sonic-yang-models/tests/yang_model_tests/yangTest.json

And test Code is in:

https://github.com/Azure/sonic-buildimage/blob/master/src/sonic-yang-models/tests/yang_model_tests/test_yang_model.py

There are 4 kinds to tests:


## Test 1: YANG Tree
### Generate YANG Tree, see no error in it.
    pyang_tree_cmd = "pyang -f tree ./yang-models/*.yang > ./yang-models/sonic_yang_tree" 
    
This Test will make sure that yang tree is generated correctly from yang models. YANG Tree should be generated as per RFC 8340. Below example shows YANG tree for interface.
```
module: sonic-interface
  +--rw sonic-interface
     +--rw INTERFACE
        +--rw INTERFACE_LIST* [port_name]
        |  +--rw port_name    -> /port:sonic-port/PORT/PORT_LIST/port_name
        |  +--rw vrf_name?    string
        +--rw INTERFACE_IPPREFIX_LIST* [port_name ip-prefix]
           +--rw port_name    -> /port:sonic-port/PORT/PORT_LIST/port_name
           +--rw ip-prefix    union
           +--rw scope?       enumeration
           +--rw family?      stypes:ip-family
```
For rest 3 kinds of test cases, below dictionary will be used which describes error strings. These strings will be part of LibYang error code for particular constraints.
```
self.defaultYANGFailure = {
    'Must': ['Must condition', 'not satisfied'],
    'InvalidValue': ['Invalid value'],
    'LeafRef': ['Leafref', 'non-existing'],
    'When': ['When condition', 'not satisfied'],
    'Pattern': ['pattern', 'does not satisfy'],
    'Mandatory': ['required element', 'Missing'],
    'Verify': ['verified'],
    'None': []
}
```
## Test 2: Successful data tree creation.
Second kind of test cases make sure that data tree is created successfully for a given config in sonic_yang.Json format.

Example:
yangTest.json will contain below example config, Note, Config should be valid and complete.
```
    "PORT_CHANNEL_TEST": {
		"sonic-portchannel:sonic-portchannel": {
			"sonic-portchannel:PORTCHANNEL": {
				"PORTCHANNEL_LIST": [{
					"portchannel_name": "PortChannel0001",
					"admin_status": "up",
					"members": [
						"Ethernet0"
					],
					"min_links": "1",
					"mtu": "9100"
				}]
			}
		},
		"sonic-port:sonic-port": {
			"sonic-port:PORT": {
				"PORT_LIST": [{
						"port_name": "Ethernet0",
						"alias": "eth0",
						"description": "Ethernet0",
						"speed": 25000,
						"mtu": 9000,
						"lanes": "65",
						"admin_status": "up"
				}]
			}
		}
	},
```
Python test file will have an entry as below which has a description field for the test and also specifies that for configuration in PORT_CHANNEL_TEST, we are expecting no error from Libyang.
```
'PORT_CHANNEL_TEST': {
                'desc': 'Configure a member port in PORT_CHANNEL table.',
                'eStr': self.defaultYANGFailure['None']
},
```
## Test3: Check constraints such as Must, Pattern, When, Mandatory etc.
Third kind of test checks the processing or error-tag of constraints such as Must, Pattern, When, Mandatory etc.

For example, below is for a field 'type ' in 'ACL table' and a test case should be written for statement 'mandatory':
```
leaf type {
	mandatory true;
	type stypes:acl_table_type;
}
```

yangTest.json may contain below example config, Where we can notice that field 'type' is missing.
```
"ACL_TABLE_MANDATORY_TYPE": {
	"sonic-acl:sonic-acl": {
		"sonic-acl:ACL_TABLE": {
			"ACL_TABLE_LIST": [{
				"ACL_TABLE_NAME": "NO-NSW-PACL-V4",
				"policy_desc": "Filter IPv4",
				"stage": "EGRESS"
			}]
		}
	}
},
```

Python test file will have an entry as below which specifies that for configuration in ACL_TABLE_MANDATORY_TYPE, we are expecting error for mandatory statement of field 'type'.
```
'ACL_TABLE_MANDATORY_TYPE': {
    'desc': 'ACL_TABLE MANDATORY TYPE FIELD.',
    'eStr': self.defaultYANGFailure['Mandatory'] + ['type', 'ACL_TABLE']
},
```

## Test4: default statement and value in Data tree.
These kind of test cases make sure that default statement are processed correctly, and the value is created in data tree even when config doesn't specifies it.

Below is an example for field 'stage' in ACL table:
```
leaf stage {
	type string {
		pattern "ingress|egress|INGRESS|EGRESS";
	}
	default "INGRESS";
}
```
yangTest.json may contain below example config, where we can notice that field 'stage' is missing.
```
"ACL_TABLE_DEFAULT_VALUE_STAGE": {
	"sonic-acl:sonic-acl": {
		"sonic-acl:ACL_TABLE": {
			"ACL_TABLE_LIST": [{
				"ACL_TABLE_NAME": "NO-NSW-PACL-V4",
				"policy_desc": "Filter IPv4",
				"type": "L3"
			}]
		}
	}
},
```	
Python test file will have an entry as below which specifies that for configuration in ACL_TABLE_DEFAULT_VALUE_STAGE, we need to Verify the value of field 'stage’ in data tree. For that we add a dictionary as below where 'verify' section will contain xpath, key and value.
```
'ACL_TABLE_DEFAULT_VALUE_STAGE': {
    'desc': 'ACL_TABLE DEFAULT VALUE FOR STAGE FIELD.',
    'eStr': self.defaultYANGFailure['Verify'],
    'verify': {'xpath': "/sonic-acl:sonic-acl/ACL_TABLE/ACL_TABLE_LIST[ACL_TABLE_NAME='NO-NSW-PACL-V4']/ACL_TABLE_NAME",
        'key': 'sonic-acl:stage',
        'value': 'INGRESS'
    }
},
```
# Python Library for Yang (PLY):

PLY library is written mainly in below 2 files, which together contains a single class.

https://github.com/Azure/sonic-buildimage/blob/master/src/sonic-yang-mgmt/sonic_yang.py 

(Wrapper on Libyang: find_data\schema_dependencies, add_node, del_node)

This part of the library can work with Any YANG model.

Below is the list of few APIs which are written on top of Libyang library:
```
_load_schema_modules_ctx
_load_data_file
_get_module_name
_get_module
_load_data_model
_print_data_mem
_save_data_file_json
_get_module_tree
_validate_data
validate_data_tree
_find_parent_data_node
_get_parent_data_xpath
_new_data_node
_find_data_node
_find_schema_node
_find_data_node_schema_xpath
_add_data_node
_merge_data
_deleteNode
_find_data_node_value
_set_data_node_value
_find_data_nodes
_find_schema_dependencies
find_data_dependencies
```
Second file contains SONiC yang models specific code which mainly does translation from config_db.json to sonic_yang.jason and vice versa.

https://github.com/Azure/sonic-buildimage/blob/master/src/sonic-yang-mgmt/sonic_yang_ext.py 

(SONiC specific functions: xlate and revXlate).

Below is the list of few APIs which are written on top of Libyang library:
```
_cropConfigDB
_extractKey
_fillLeafDict
_fillSteps
_createLeafDict
_findYangTypedValue
_yangConvert
_xlateList
_xlateListInContainer
_xlateContainerInContainer
_xlateContainer
_xlateConfigDBtoYang
_xlateConfigDB
_createKey
_revFindYangTypedValue
_revYangConvert
_revXlateList
_revXlateListInContainer
_revXlateContainerInContainer
_revXlateContainer
_revXlateYangtoConfigDB
_revXlateConfigDB
_findYangList
findXpathPortLeaf
findXpathPort
_findXpathList
loadData
getData
deleteNode
```

# Python Library for Yang Tests:
Below file contanins test cases for PLY:

https://github.com/Azure/sonic-buildimage/blob/master/src/sonic-yang-mgmt/tests/libyang-python-tests/test_sonic_yang.py

One snapshot of SONiC YANG models is used for Libyang Wrapper testing.
List of Test functions:
```
test_load_yang_model_files
test_load_invalid_model_files
test_load_yang_model_dir
test_load_yang_model_data
test_load_data_file
test_validate_data_tree
test_find_node
test_add_node
test_find_data_node_value
test_delete_node
test_set_datanode_value
test_find_members
test_get_parent_data_xpath
test_find_data_node_schema_xpath
test_find_data_dependencies
test_find_schema_dependencies
test_merge_data_tree
test_xlate_rev_xlate()
```

# Development and Validation of new Yang Models.
How to make sure YANG models are written or updated for any new table or new field in existing table of config_db:

As part of SONiC YANG workgroup below measure will be taken while review is done for HLD of new SONiC feature:

a.) Any change to ConfigDB schema MUST include YANG model schema change.

b.) PR covering YANG model update MUST be reviewed and approved by YANG subgroup.

c.) HLD approval can be BLOCKED if YANG model is not reviewed and approved by YANG subgroup.

Build time tests will check for 

a.) Existence of YANG models and 

b.) Validation of YANG models as per SONiC YANG Models Guidelines using below steps:

-- Developer will have to update new config in 

https://github.com/Azure/sonic-buildimage/tree/master/src/sonic-yang-models/sample_config_db.json

-- At build time, Package sonic-yang-mgmt internally will try to load SONiC config from this file 

a.) by converting the config as per RFC 7951 using YANG Models, 

b.) by creating data tree using new YANG models and 

c.) by validating config against YANG models. 

Successful execution of these steps can be treated as validation of new Yang models.