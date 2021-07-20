# Enhancements to Optical Transceiver Information Model
## Introduction
Currently, xcvrd parses the EEPROMs of optical media transceivers, and builds a Python data structure, a dictionary, containing transceiver information.

This design is an effort to enhance and standardize the representation of transceiver information.
## Transceiver information
### Parsing
Xcvrd shall parse optical media transceiver EEPROMs, by means of core, platform-independent code, and also using an optional plugin provided by a platform implementation, to handle possibly vendor-proprietary information encoded in the EEPROM.

The architecture of the xcvrd code is beyond the scope this document; that is the HOW of obtaining transceiver information, this document concentrates on the WHAT (syntax and semantics) of transceiver information.

This design is intended to standardize the minimum set of transceiver information; platform-dependent plug-ins may provide additional information, but all fields in this document must be present for all platforms.
### Goals
The goal is to have xcvrd build a Python dictionary, containing all static transceiver information (i.e. transceiver information that does not change over time) for an installed transceiver, information that is needed to:
- properly configure the NPU for the installed transceiver, e.g. set the NPU interface type to "fiber" or "copper";
- provide information for making a platform-dependent, port-dependent and transceiver-dependent lookup of port configuration information to be passed to the NPU, e.g. look up port pre-emphasis value(s);
- provide information for determining possible port configurations, and validating requested port configurations, e.g. determining what, if any, breakout modes are possible on a port; and
- display transceiver information to the user.

The format of this dictionary, i.e. the member fields, and the contents of all those fields, shall be standardized, and platform-independent, so that the code to configure the NPU and display information to the user can also be platform-independent.

This dictionary, once created by xcvrd, is published to the Redis DB, as a record in the TRANSCEIVER_INFO table.  NPU-configuration and CLI code will obtain transceiver information from there.
### Current format
An example of the transceiver information dictionary constructed by xcvrd is as follows:
~~
{
	"Connector": "No separable connector", 
	"cable_length": "2", 
	"cable_type": "Length Cable Assembly(m)", 
	"encoding": "64B66B", 
	"ext_identifier": "Power Class 1(1.5W max)", 
	"ext_rateselect_compliance": "QSFP+ Rate Select Version 1", 
	"hardwarerev": "A0", 
	"manufacturename": "DELL", 
	"modelname": "76V43", 
	"nominal_bit_rate": "255", 
	"serialnum": "CN0LXD0096H3A36", 
	"specification_compliance": "{'10/40G Ethernet Compliance Code': '40GBASE-CR4'}", 
	"type": "QSFP28 or later", 
	"type_abbrv_name": "QSFP28", 
	"vendor_date": "2019-06-17 ", 
	"vendor_oui": "3c-18-a0"
}
~~
### Proposed format
The intention is to maintain existing field names and their contents, for backward-compatibility.  New fields shall be added.

The following fields shall be added to the dictionary:

<h4>cable_breakout</h4>
<table style="border-style:none;">
	<tr>
		<td valign="top">Description</td>
		<td>Breakout of attached cable (if any)</td>
	</tr>
	<tr>
		<td valign="top">Value(s)</td>
		<td>string; one of '1x1', '1x2', '1x4', '2x2', etc.
        <br>
        Contains an empty string ('') when no adapter installed</td>
	</tr>
	<tr>
		<td valign="top">Needed by</td>
		<td>Validation of port breakout configurations, i.e. whether or not transceiver supports breakout</td>
	</tr>
</table>
<h4>cable_length_detailed</h4>
<table style="border-style:none;">
	<tr>
		<td valign="top">Description</td>
		<td>Length, in metres, of directly-attached cable (if any)</td>
	</tr>
	<tr>
		<td valign="top">Value(s)</td>
		<td>string; length of cable, floating point value with 1 decimal place, as string, e.g. '10.0'
        <br>
        Contains an empty string ('') when adapter has no directly-attached cable, or no adapter installed</td>
	</tr>
	<tr>
		<td valign="top">Needed by</td>
		<td>Layer-1 configuration lookup, e.g preemphasis</td>
	</tr>
	<tr>
		<td valign="top">Notes</td>
		<td>Complements existing cable_length field, can represent lengths < 1 m or fractional lengths</td>
	</tr>
</table>
<h4>cable_type</h4>
<table style="border-style:none;">
	<tr>
		<td valign="top">Description</td>
		<td>Type of directly-attached cable (if any)</td>
	</tr>
	<tr>
		<td valign="top">Value(s)</td>
		<td>string, one of "DAC", "RJ45", "FIBER", "AOC", "ACC"
        <br>
		Contains empty string ('') when adapter has no directly-attached cable, or no adapter is installed
		</td>
	</tr>
	<tr>
		<td valign="top">Needed by</td>
		<td>Layer-1 configuration lookup, e.g NPU interface type</td>
	</tr>
</table>
<h4>display_name</h4>
<table style="border-style:none;">
	<tr>
		<td valign="top">Description</td>
		<td>String suitable for displaying description of transceiver for user</td>
	</tr>
	<tr>
		<td valign="top">Value(s)</td>
		<td>string
        <br>
		Contains empty string ('') when no adapter installed
		</td>
	</tr>
	<tr>
		<td valign="top">Needed by</td>
		<td>CLI show command(s)</td>
	</tr>
	<tr>
		<td valign="top">Notes</td>
		<td>Rather than the CLI code attempt to construct descriptive string for an installed transceiver, from the information in the TRANSCEIVER_TABLE, xcvrd can construct a short descriptive string for use by CLI show commands.  Since transceiver "knowledge" is inside xcvrd, it is reasonable for it to compose the displayable descriptive string, rather than the CLI.</td>
	</tr>
	<tr>
		<td valign="top">Examples</td>
		<td>"SFP+ 10GBASE-DWDM-TUNABLE"
        <br>
        "QSFP28 100GBASE-CR4-0.5M"</td>
	</tr>
</table>
<h4>lane_count</h4>
<table style="border-style:none;">
	<tr>
		<td valign="top">Description</td>
		<td>Number of lanes</td>
	</tr>
	<tr>
		<td valign="top">Value(s)</td>
		<td>string; representing a positive integer, 1..N
        <br>
		Contains empty string ('') when no adapter installed
		</td>
	</tr>
	<tr>
		<td valign="top">Needed by</td>
		<td>Validation of port breakout configurations, i.e. whether or not transceiver supports breakout</td>
	</tr>
</table>
<h4>media_interface</h4>
<table style="border-style:none;">
	<tr>
		<td valign="top">Description</td>
		<td>Media interface</td>
	</tr>
	<tr>
		<td valign="top">Value</td>
		<td>string, one of "BIDI", "BX", "CLR", "CR", "CWDM", "CX", "DR", "DWDM", "ER", "FR", "FX", "LR", "LRM", "LX", "PSM", "PX", "SR", "SWDM", "SX", "T", "WDM"
        <br>
		Contains empty string ('') when no adapter installed
		</td>
	</tr>
	<tr>
		<td valign="top">Needed by</td>
		<td>Layer-1 configuration lookup, e.g preemphasis</td>
	</tr>
</table>
<h4>form_factor</h4>
<table style="border-style:none;">
	<tr>
		<td valign="top">Description</td>
		<td>Detailed transceiver type, i.e. form factor</td>
	</tr>
	<tr>
		<td valign="top">Value(s)</td>
		<td>string, one of "SFP", "SFP+", "SFP28", "SFP56-DD", "QSFP+", "QSFP28", "QSFP28-DD", "QSFP56-DD", "RJ45"
        <br>
        Contains empty string ('') when no adapter installed
</td>
	</tr>
	<tr>
		<td valign="top">Needed by</td>
		<td>Layer-1 configuration lookup, e.g preemphasis</td>
	</tr>
	<tr>
		<td valign="top">Notes</td>
		<td>This field is intended as a replacement for the existing "type_abbrv_name" field.  That field does not fully discriminate a transceiver "type".  For example, a QSFP-DD transceiver may be a QSFP28-DD or QSFP56-DD.  Also, other types, such as "SFP28", are missing.<br/>Rather than introduce new values, such as "SFP28", "QSFP-DD28" and "QSFP-DD56" for the existing field, and deprecating old values, such as "QSFP-DD", a new field will be introduced, that fully discriminates a "type", and the old field shall be deprecated.</td>
	</tr>
</table>

## Implementation Note
Details of how an adapter EEPROM is parsed, e.g. the reading of the EEPROM bytes from the adapter, the decoding of EEPROM bytes, according to various standards, memory maps (DD or otherwise), etc. is the responsibility of the code implementing the Python data model described here, and code previously implemented in xcvrd; implementation details are not discussed in this document. 
