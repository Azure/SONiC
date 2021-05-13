## CMIS and C-CMIS support for ZR on SONiC

### Overview
Common Management Interface Specification (CMIS) is defined for pluggables or on-board modules to communicate with the registers [CMIS v4.0](http://www.qsfp-dd.com/wp-content/uploads/2019/05/QSFP-DD-CMIS-rev4p0.pdf). With a clear difinition of these registers, modules can set the configurations or get the status, to achieve the basic level of monitor and control. 

CMIS is widely used on modules based on a Two-Wire-Interface (TWI), including QSFP-DD, OSFP, COBO and QSFP modules. However, new requirements emerge with the introduction of coherent optical modules, such as 400G ZR. 400G ZR is the first type of modules to require definitions on coherent optical specifications, a field CMIS does not touch on. The development of C(coherent)-CMIS aims to solve this issue [C-CMIS v1.1](https://www.oiforum.com/wp-content/uploads/OIF-C-CMIS-01.1.pdf). It is based on CMIS but incroporates more definitions on registers in the extended space, regarding the emerging demands on coherent optics specifications.

The scope of this work is to develop APIs for both CMIS and C-CMIS to support 400G ZR modules on SONiC.

The rest of the article will discuss the following items:

- Layered architecture to access registers
- Definition on CMIS and C-CMIS registers
- Method to read from and write to registers
- High level functions
- Module firmware upgrade using command data block (CDB)

### Layered architecture to access registers
          ---------------------------
         |   High level functions    |
          ---------------------------
                /\            ||            
                ||            \/
             ---------------------
            | Decode       Encode |
             ---------------------
                /\            ||            
                ||            \/
            ------------------------
           | read_reg     write_reg |
            ------------------------               
                /\            ||            
                ||            \/
             ---------------------
            |   Module registers  |
             ---------------------           
                

### Definition on CMIS and C-CMIS registers
-  Memory structure and mapping

The host addressable memory starts with a lower memory of 128 bytes that occupy address byte 0-127. 
Then it starts from Page 0. Each page has 128 bytes and the first byte of each page starts with an offset of 128.
Therefore, the address of a byte with *page* and *offset* is *page* * 128 + *offset*.

-  Module general information pages (Page 0h - 1Fh, CMIS). See Figure 8-1 and Figure 8-2 in [CMIS](http://www.qsfp-dd.com/wp-content/uploads/2019/05/QSFP-DD-CMIS-rev4p0.pdf)

Important pages containing module general information:

|Address|Page Description|Type|
|-------|----------------|----|
|00h|Administrative Information|RO|
|01h|Advertising|RO|
|02h|Threshold Information|RO|
|04h|Laser Capabilities Advertising|RO|
|10h|Lane and Datapath Configuration|RW|
|11h|Lane and Datapath Status|RO|
|12h|Tunable Laser Control and Status|mixed|


Sample code to define registers with module general information:

```
SFF8024_IDENTIFIER = {
          'PAGE': 0x00,
          'OFFSET': 0,
          'SIZE': 1,
          'TYPE': 'B'
}
```
-  Versatile Diagnostics Monitor (VDM) pages (Page 20h - 2Fh, CMIS and C-CMIS)

VDM Pages. See Table 8-95 in [CMIS](http://www.qsfp-dd.com/wp-content/uploads/2019/05/QSFP-DD-CMIS-rev4p0.pdf)

|Address|Page Description|Type|
|-------|----------------|----|
|20h|VDM Observable Descriptors 1| RO|
|21h|VDM Observable Descriptors 2| RO|
|22h|VDM Observable Descriptors 3| RO|
|23h|VDM Observable Descriptors 4| RO|
|24h|VDM Samples 1| RO|
|25h|VDM Samples 2| RO|
|26h|VDM Samples 3| RO|
|27h|VDM Samples 4| RO|
|28h|VDM Thresholds 1| RO|
|29h|VDM Thresholds 2| RO|
|2Ah|VDM Thresholds 3| RO|
|2Bh|VDM Thresholds 4| RO|
|2Ch|VDM Flags|RO/COR|
|2Dh|VDM Masks|RW|


Note that VDM ID 1-24 are defined in CMIS. VDM ID starting from 128 are defined in C-CMIS. Below is sample code to define registers with VDM information. 
We use ```get_VDM``` function to get VDM items and their thresholds, which will be introduced later in this document [here](#get-vdm-related-information).

```
VDM_TYPE = {
          # VDM_ID: [VDM_NAME, DATA_TYPE, SCALE]
          1: ['Laser Age [%]', 'U16', 1],
          2: ['TEC Current [%]', 'S16', 100.0/32767],
          3: ['Laser Frequency Error [MHz]', 'S16', 10],
          4: ['Laser Temperature [C]', 'S16', 1.0/256],
          5: ['eSNR Media Input [dB]', 'U16', 1.0/256],
          6: ['eSNR Host Input [dB]', 'U16', 1.0/256],
          7: ['PAM4 Level Transition Parameter Media Input [dB]', 'U16', 1.0/256],
          8: ['PAM4 Level Transition Parameter Host Input [dB]', 'U16', 1.0/256],
          9: ['Pre-FEC BER Minimum Media Input', 'F16', 1], 
          10: ['Pre-FEC BER Minimum Host Input', 'F16', 1], 
          11: ['Pre-FEC BER Maximum Media Input', 'F16', 1], 
          12: ['Pre-FEC BER Maximum Host Input', 'F16', 1], 
          13: ['Pre-FEC BER Average Media Input', 'F16', 1], 
          14: ['Pre-FEC BER Average Host Input', 'F16', 1], 
          15: ['Pre-FEC BER Current Value Media Input', 'F16', 1],
          16: ['Pre-FEC BER Current Value Host Input', 'F16', 1],
          17: ['Errored Frames Minimum Media Input', 'F16', 1], 
          18: ['Errored Frames Minimum Host Input', 'F16', 1], 
          19: ['Errored Frames Maximum Media Input', 'F16', 1], 
          20: ['Errored Frames Minimum Host Input', 'F16', 1], 
          21: ['Errored Frames Average Media Input', 'F16', 1], 
          22: ['Errored Frames Average Host Input', 'F16', 1], 
          23: ['Errored Frames Current Value Media Input', 'F16', 1], 
          24: ['Errored Frames Current Value Host Input', 'F16', 1],
          128: ['Modulator Bias X/I [%]', 'U16', 100.0/65535],
          129: ['Modulator Bias X/Q [%]', 'U16', 100.0/65535],
          130: ['Modulator Bias Y/I [%]', 'U16', 100.0/65535],
          131: ['Modulator Bias Y/Q [%]', 'U16', 100.0/65535],
          132: ['Modulator Bias X_Phase [%]', 'U16', 100.0/65535],
          133: ['Modulator Bias Y_Phase [%]', 'U16', 100.0/65535],
          134: ['CD high granularity, short link [ps/nm]', 'S16', 1], 
          135: ['CD low granularity, long link [ps/nm]', 'S16', 20],
          136: ['DGD [ps]', 'U16', 0.01],
          137: ['SOPMD [ps^2]', 'U16', 0.01],
          138: ['PDL [dB]', 'U16', 0.1],
          139: ['OSNR [dB]', 'U16', 0.1],
          140: ['eSNR [dB]', 'U16', 0.1],
          141: ['CFO [MHz]', 'S16', 1],
          142: ['EVM_modem [%]', 'U16', 100.0/65535],
          143: ['Tx Power [dBm]', 'S16', 0.01],
          144: ['Rx Total Power [dBm]', 'S16', 0.01],
          145: ['Rx Signal Power [dBm]', 'S16', 0.01],
          146: ['SOP ROC [krad/s]', 'U16', 1],
          147: ['MER [dB]', 'U16', 0.1]
}
```
-  C-CMIS related pages (Page 30h - 4Fh, C-CMIS)

See [C-CMIS v1.1](https://www.oiforum.com/wp-content/uploads/OIF-C-CMIS-01.1.pdf)

|Address|Page Description|Type|
|-------|----------------|----|
|30h|Media Lane Configurable Thresholds|RW|
|31h|Media Lane Provisioning|RW|
|32h|Media Lane Masks|RW|
|33h|Media Lane Flags|RO/COR|
|34h|Media Lane FEC PM|RO|
|35h|Media Lane Link PM|RO|
|38h|Host Interface Configuration|RW|
|3Ah|Host Interface PM|RO|
|3Bh|Host Interface Flags and Masks|mixed|
|41h|RX Power Advertisement and Configurable Thresholds|RO|
|42h|PM Advertisement|RO|
|43h|Media Lane Provisioning Advertisement|RO|

### Method to read from and write to registers

#### Read and write registers
- read_reg
- write_reg

Read and write registers use vendor provided functions to access the bottom layer registers with dictionaries defined Sample code to read and write registers with vendor provided functoins. As mentioned above, the address of a byte with *page* and *offset* is *page* * 128 + *offset*. Below is sample code to read and write registers.

```
import sonic_platform.platform
import sonic_platform_base.sonic_sfp.sfputilhelper
platform_chassis = sonic_platform.platform.Platform().get_chassis()
PAGE_SIZE = 128

def read_reg(port, page, offset, size):
    return platform_chassis.get_sfp(port).read_eeprom(page*PAGE_SIZE + offset,size)

def write_reg(port, page, offset, size, write_raw):
    platform_chassis.get_sfp(port).write_eeprom(page*PAGE_SIZE + offset,size,write_raw)
```
#### Encoding and decoding raw data
- read_reg_from_dict
- write_reg_from_dict

Read and write registers from dictionary use dictionaries defined [here](#definition-on-cmis-and-c-cmis-registers) and calls read and write register function [here](#read-and-write-registers). We use Python built-in library ```struct``` to encode and decode the raw data in bytearray form to meaningful values. Sample code to decode and encode from/to rawdata in registers:

```
def read_reg_from_dict(port, Dict): 
    read_raw = read_reg(port, page = Dict['PAGE'], offset = Dict['OFFSET'], size = Dict['SIZE'])
    read_buffer = struct.unpack(Dict['TYPE'], read_raw)
    if len(read_buffer) == 1:
        return read_buffer[0]
    else:
        return read_buffer

def write_reg_from_dict(port, Dict, write_buffer):
    write_raw = struct.pack(Dict['TYPE'], write_buffer)
    write_reg(port, page = Dict['PAGE'], offset = Dict['OFFSET'], size = Dict['SIZE'], write_raw = write_raw)
```

### High level functions

#### Get module basic information
- get_module_type

```
def get_module_type(port):
    module_type = read_reg_from_dict(port, Page00h_Lower.SFF8024_IDENTIFIER)
    # 400ZR specific: 18h, QSFP-DD
    return Data_Type_Dict.MODULE_TYPE_DICT[module_type]
```

- get_module_status

```
def get_module_status(port):
    # Bit 3-1:      Module state. 
    # 000b          -
    # 001b          ModuleLowPwr
    # 010b          ModulePwrUp
    # 011b          ModuleReady This is the only state reported by flat memory modules
    # 100b          ModulePwrDn
    # 101b          Fault
    # 110b          -
    # 111b          -

    # Bit 0: Interrupt status. Status of Interrupt output (inverted logic hardware signal)
    # 0b: Interrupt asserted
    # 1b: Interrupt not asserted (default)
    module_status_raw = read_reg_from_dict(port, Page00h_Lower.MODULE_STATE)
    module_status = (module_status_raw>>1) & 0x7
    return Data_Type_Dict.MODULE_STATUS_DICT[module_status]
```

- get_module_vendor

```
def get_module_vendor(port):
    module_vendor = read_reg_from_dict(port, Page00h_Upper.VENDOR_NAME)
    return module_vendor
```
- get_module_part_number

```
def get_module_part_number(port):
    module_part_number = read_reg_from_dict(port, Page00h_Upper.VENDOR_PN)
    return module_part_number
```
- get_module_serial_number

```
def get_module_serial_nubmer(port):
    module_serial_number = read_reg_from_dict(port, Page00h_Upper.VENDOR_SN)
    return module_serial_number
```
- get_datapath_lane_status

```
def get_datapath_lane_status(port):
    LANE_NUMBER = 8
    datapath_lane_status = [0] * LANE_NUMBER

    # Bit 7-4: DataPathStateHostLane2 Data Path State Indicator, as observed on host lane 2
    # Bit 3-0: DataPathStateHostLane1 Data Path State Indicator, as observed on host lane 1
    # Bit 7-4: DataPathStateHostLane4 Data Path State Indicator, as observed on host lane 4
    # Bit 3-0: DataPathStateHostLane3 Data Path State Indicator, as observed on host lane 3
    # Bit 7-4: DataPathStateHostLane6 Data Path State Indicator, as observed on host lane 6
    # Bit 3-0: DataPathStateHostLane5 Data Path State Indicator, as observed on host lane 5
    # Bit 7-4: DataPathStateHostLane8 Data Path State Indicator, as observed on host lane 8
    # Bit 3-0: DataPathStateHostLane7 Data Path State Indicator, as observed on host lane 7
    datapath_state_1 = read_reg_from_dict(port, Page11h.DATA_PATH_STATE_HOST_1)
    datapath_state_2 = read_reg_from_dict(port, Page11h.DATA_PATH_STATE_HOST_2)
    datapath_state_3 = read_reg_from_dict(port, Page11h.DATA_PATH_STATE_HOST_3)
    datapath_state_4 = read_reg_from_dict(port, Page11h.DATA_PATH_STATE_HOST_4)

    # Data path state
    # Encoding      State
    # 0h            Reserved
    # 1h            DataPathDeactivated
    # 2h            DataPathInit
    # 3h            DataPathDeinit
    # 4h            DataPathActivated
    # 5h            DataPathTxTurnOn
    # 6h            DataPathTxTurnOff
    # 7h            DataPathInitialized
    # 8h-Fh         Reserved

    datapath_lane_status[0] = datapath_state_1 & 0xF
    datapath_lane_status[1] = (datapath_state_1>>4) & 0xF
    datapath_lane_status[2] = datapath_state_2 & 0xF
    datapath_lane_status[3] = (datapath_state_2>>4) & 0xF
    datapath_lane_status[4] = datapath_state_3 & 0xF
    datapath_lane_status[5] = (datapath_state_3>>4) & 0xF
    datapath_lane_status[6] = datapath_state_4 & 0xF
    datapath_lane_status[7] = (datapath_state_4>>4) & 0xF
    return Data_Type_Dict.DATA_PATH_STATUS_DICT[datapath_lane_status[0]]
```
- get_module_case_temp

```
def get_module_case_temp(port):
    # Scaled with 1/256 degree Celsius increments. Unit in deg C
    MODULE_CASE_TEMP_SCALE = 1.0/256
    module_case_temp = read_reg_from_dict(port, Page00h_Lower.MODULE_CASE_TEMP) * MODULE_CASE_TEMP_SCALE
    return module_case_temp
```
- get_supply_3v3

```
def get_supply_3v3(port):
    # Scaled with 100 uV increments. Unit converted to Volt
    SUPPLY_3V3_SCALE = 0.0001
    supply_3v3 = read_reg_from_dict(port, Page00h_Lower.SUPPLY_3V3) * SUPPLY_3V3_SCALE
    return supply_3v3
```
- get_laser_temp

```
def get_laser_temp(port):
    aux_monitor_ad = read_reg_from_dict(port, Page01h.MODULE_CHARACTERISTICS_MISC_1)
    # Bit 1: Aux2MonitorType
    # 0b: Aux 2 monitor monitors Laser Temperature
    # 1b: Aux 2 monitor monitors TEC current    
    Aux2MonitorType = (aux_monitor_ad>>1) & 0x1
    # Bit 2: Aux3MonitorType
    # 0b: Aux 3 monitor monitors Laser Temperature
    # 1b: Aux 3 monitor monitors Vcc2
    Aux3MonitorType = (aux_monitor_ad>>2) & 0x1

    # laser temp scaled with 1/256 degree Celsius increments. Unit in deg C
    # laser temp from Aux3
    LASER_TEMP_SCALE = 1.0/256
    if Aux2MonitorType and not Aux3MonitorType: 
        laser_temp = read_reg_from_dict(port, Page00h_Lower.AUX3MONITOR) * LASER_TEMP_SCALE
    # laser temp from Aux2
    elif not Aux2MonitorType and Aux3MonitorType: 
        laser_temp = read_reg_from_dict(port, Page00h_Lower.AUX2MONITOR) * LASER_TEMP_SCALE
    # laser temp monitor not supported
    else:
        laser_temp = None
    return laser_temp
```
- get_tuning_status

```
def get_tuning_status(port):
    tuning_status = read_reg_from_dict(port, Page12h.TX_TUNING_STATUS_LANE_1) & 0x3
    return Data_Type_Dict.LASER_TUNING_STATUS_DICT[tuning_status]
```
- get_laser_freq

```
def get_laser_freq(port):
    # Unit in MHz
    laser_freq = read_reg_from_dict(port, Page12h.TX_CURRENT_LASER_FREQUENCY_LANE_1)
    return laser_freq
```
- get_TX_configured_power

```
def get_TX_configured_power(port):
    # configured TX power value
    # Scaled with 0.01 dBm. Unit in dBm
    TX_POWER_SCALE = 0.01
    tx_configured_power = read_reg_from_dict(port, Page12h.TX_TARGET_OUTPUT_POWER_LANE_1) * TX_POWER_SCALE
```

#### Get VDM related information
- get_VDM

```get_VDM_page``` function uses ```VDM_TYPE``` dictionary above. It parses all the VDM items defined in the dictionary within a certain VDM page and returns both VDM monitor values and four threshold values related to this VDM item. ```get_VDM``` function combines VDM items from all VDM pages.

```
PAGE_SIZE = 128
PAGE_OFFSET = 128
THRSH_SPACING = 8
VDM_SIZE = 2

def get_F16(value):
    scale_exponent = (value >> 11) & 0x1f
    mantissa = value & 0x7ff
    result = mantissa*10**(scale_exponent-24)
    return result

def get_VDM_page(port, page):
    if page not in [0x20, 0x21, 0x22, 0x23]:
        raise ValueError('Page not in VDM Descriptor range!')
    VDM_descriptor = struct.unpack(f'{PAGE_SIZE}B', read_reg(port, page, PAGE_OFFSET, PAGE_SIZE))
    # Odd Adress VDM observable type ID, real-time monitored value in Page + 4
    VDM_typeID = VDM_descriptor[1::2]
    # Even Address
    # Bit 7-4: Threshold set ID in Page + 8, in group of 8 bytes, 16 sets/page
    # Bit 3-0: n. Monitored lane n+1 
    VDM_lane = [(elem & 0xf) for elem in VDM_descriptor[0::2]]
    VDM_thresholdID = [(elem>>4) for elem in VDM_descriptor[0::2]]
    VDM_valuePage = page+4
    VDM_thrshPage = page+8
    VDM_Page_data = {}
    for index, typeID in enumerate(VDM_typeID):
        if typeID not in Data_Type_Dict.VDM_TYPE:
            continue
        else:
            vdm_info_dict = Data_Type_Dict.VDM_TYPE[typeID]
            scale = vdm_info_dict[2]
            thrshID = VDM_thresholdID[index]
            vdm_value_raw = read_reg(port, VDM_valuePage, PAGE_OFFSET+VDM_SIZE*index, VDM_SIZE)
            vdm_thrsh_high_alarm_raw = read_reg(port, VDM_thrshPage, PAGE_OFFSET + THRSH_SPACING * thrshID, VDM_SIZE)
            vdm_thrsh_low_alarm_raw = read_reg(port, VDM_thrshPage, PAGE_OFFSET + THRSH_SPACING * thrshID + 2, VDM_SIZE)
            vdm_thrsh_high_warn_raw = read_reg(port, VDM_thrshPage, PAGE_OFFSET + THRSH_SPACING * thrshID + 4, VDM_SIZE)
            vdm_thrsh_low_warn_raw = read_reg(port, VDM_thrshPage, PAGE_OFFSET + THRSH_SPACING * thrshID + 6, VDM_SIZE)
            if vdm_info_dict[1] == 'S16':
                vdm_value = struct.unpack('>h',vdm_value_raw)[0] * scale
                vdm_thrsh_high_alarm = struct.unpack('>h', vdm_thrsh_high_alarm_raw)[0] * scale
                vdm_thrsh_low_alarm = struct.unpack('>h', vdm_thrsh_low_alarm_raw)[0] * scale
                vdm_thrsh_high_warn = struct.unpack('>h', vdm_thrsh_high_warn_raw)[0] * scale
                vdm_thrsh_low_warn = struct.unpack('>h', vdm_thrsh_low_warn_raw)[0] * scale
            elif vdm_info_dict[1] == 'U16':
                vdm_value = struct.unpack('>H',vdm_value_raw)[0] * scale
                vdm_thrsh_high_alarm = struct.unpack('>H', vdm_thrsh_high_alarm_raw)[0] * scale
                vdm_thrsh_low_alarm = struct.unpack('>H', vdm_thrsh_low_alarm_raw)[0] * scale
                vdm_thrsh_high_warn = struct.unpack('>H', vdm_thrsh_high_warn_raw)[0] * scale
                vdm_thrsh_low_warn = struct.unpack('>H', vdm_thrsh_low_warn_raw)[0] * scale
            elif vdm_info_dict[1] == 'F16':
                vdm_value_int = struct.unpack('>H',vdm_value_raw)[0]
                vdm_value = get_F16(vdm_value_int)
                vdm_thrsh_high_alarm_int = struct.unpack('>H', vdm_thrsh_high_alarm_raw)[0]
                vdm_thrsh_low_alarm_int = struct.unpack('>H', vdm_thrsh_low_alarm_raw)[0]
                vdm_thrsh_high_warn_int = struct.unpack('>H', vdm_thrsh_high_warn_raw)[0]
                vdm_thrsh_low_warn_int = struct.unpack('>H', vdm_thrsh_low_warn_raw)[0]
                vdm_thrsh_high_alarm = get_F16(vdm_thrsh_high_alarm_int)
                vdm_thrsh_low_alarm = get_F16(vdm_thrsh_low_alarm_int)
                vdm_thrsh_high_warn = get_F16(vdm_thrsh_high_warn_int)
                vdm_thrsh_low_warn = get_F16(vdm_thrsh_low_warn_int)
            else:
                continue

        if vdm_info_dict[0] not in VDM_Page_data:
            VDM_Page_data[vdm_info_dict[0]] = {VDM_lane[index]+1: [vdm_value,
                                                                   vdm_thrsh_high_alarm,
                                                                   vdm_thrsh_low_alarm,
                                                                   vdm_thrsh_high_warn,
                                                                   vdm_thrsh_low_warn]}
        else:
            VDM_Page_data[vdm_info_dict[0]][VDM_lane[index]+1] = [vdm_value,
                                                                  vdm_thrsh_high_alarm,
                                                                  vdm_thrsh_low_alarm,
                                                                  vdm_thrsh_high_warn,
                                                                  vdm_thrsh_low_warn]
    return VDM_Page_data

def get_VDM(port):
    vdm_page_supported_raw = read_reg_from_dict(port, Page2Fh.VDM_SUPPORT) & 0x3
    vdm_page_supported = Data_Type_Dict.VDM_SUPPORTED_PAGE[vdm_page_supported_raw]
    VDM = {}
    # Bit 7, freeze all PMs for reporting
    write_reg_from_dict(port, Page2Fh.FREEZE_REQUEST, 128)
    time.sleep(1)
    for page in vdm_page_supported:
        VDM_current_page = get_VDM_page(port, page)
        VDM.update(VDM_current_page)
    write_reg_from_dict(port, Page2Fh.FREEZE_REQUEST, 0)
    return VDM
```

#### Get C-CMIS PM
- get_PM

Sample code to read C-CMIS defined PMs:

```
def get_PM(port):

    write_reg_from_dict(port, Page2Fh.FREEZE_REQUEST, 128)
    time.sleep(1)

    PM_dict = {}
    rx_bits_pm = read_reg_from_dict(port, Page34h.RX_BITS_PM)
    rx_bits_subint_pm = read_reg_from_dict(port, Page34h.RX_BITS_SUB_INTERVAL_PM)
    rx_corr_bits_pm = read_reg_from_dict(port, Page34h.RX_CORR_BITS_PM)
    rx_min_corr_bits_subint_pm = read_reg_from_dict(port, Page34h.RX_MIN_CORR_BITS_SUB_INTERVAL_PM)
    rx_max_corr_bits_subint_pm = read_reg_from_dict(port, Page34h.RX_MAX_CORR_BITS_SUB_INTERVAL_PM)

    if (rx_bits_subint_pm != 0) and (rx_bits_pm != 0):
        PM_dict['preFEC_BER_cur'] = rx_corr_bits_pm*1.0/rx_bits_pm
        PM_dict['preFEC_BER_min'] = rx_min_corr_bits_subint_pm*1.0/rx_bits_subint_pm
        PM_dict['preFEC_BER_max'] = rx_max_corr_bits_subint_pm*1.0/rx_bits_subint_pm

    rx_frames_pm = read_reg_from_dict(port, Page34h.RX_FRAMES_PM)
    rx_frames_subint_pm = read_reg_from_dict(port, Page34h.RX_FRAMES_SUB_INTERVAL_PM)
    rx_frames_uncorr_err_pm = read_reg_from_dict(port, Page34h.RX_FRAMES_UNCORR_ERR_PM)
    rx_min_frames_uncorr_err_subint_pm = read_reg_from_dict(port, Page34h.RX_MIN_FRAMES_UNCORR_ERR_SUB_INTERVAL_PM)
    rx_max_frames_uncorr_err_subint_pm = read_reg_from_dict(port, Page34h.RX_MIN_FRAMES_UNCORR_ERR_SUB_INTERVAL_PM)

    if (rx_frames_subint_pm != 0) and (rx_frames_pm != 0):
        PM_dict['preFEC_uncorr_frame_ratio_cur'] = rx_frames_uncorr_err_pm*1.0/rx_frames_subint_pm
        PM_dict['preFEC_uncorr_frame_ratio_min'] = rx_min_frames_uncorr_err_subint_pm*1.0/rx_frames_subint_pm
        PM_dict['preFEC_uncorr_frame_ratio_max'] = rx_max_frames_uncorr_err_subint_pm*1.0/rx_frames_subint_pm


    PM_dict['rx_cd_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_CD_PM)
    PM_dict['rx_cd_min'] = read_reg_from_dict(port, Page35h.RX_MIN_CD_PM)
    PM_dict['rx_cd_max'] = read_reg_from_dict(port, Page35h.RX_MAX_CD_PM)

    PM_dict['rx_dgd_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_DGD_PM)*Data_Type_Dict.PM_SCALE['RX_DGD_SCALE_PS']
    PM_dict['rx_dgd_min'] = read_reg_from_dict(port, Page35h.RX_MIN_DGD_PM)*Data_Type_Dict.PM_SCALE['RX_DGD_SCALE_PS']
    PM_dict['rx_dgd_max'] = read_reg_from_dict(port, Page35h.RX_MAX_DGD_PM)*Data_Type_Dict.PM_SCALE['RX_DGD_SCALE_PS']

    PM_dict['rx_sopmd_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_SOPMD_PM)*Data_Type_Dict.PM_SCALE['RX_SOPMD_SCALE_PS2']
    PM_dict['rx_sopmd_min'] = read_reg_from_dict(port, Page35h.RX_MIN_SOPMD_PM)*Data_Type_Dict.PM_SCALE['RX_SOPMD_SCALE_PS2']
    PM_dict['rx_sopmd_max'] = read_reg_from_dict(port, Page35h.RX_MAX_SOPMD_PM)*Data_Type_Dict.PM_SCALE['RX_SOPMD_SCALE_PS2']

    PM_dict['rx_pdl_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_PDL_PM)*Data_Type_Dict.PM_SCALE['RX_PDL_SCALE_DB']
    PM_dict['rx_pdl_min'] = read_reg_from_dict(port, Page35h.RX_MIN_PDL_PM)*Data_Type_Dict.PM_SCALE['RX_PDL_SCALE_DB']
    PM_dict['rx_pdl_max'] = read_reg_from_dict(port, Page35h.RX_MAX_PDL_PM)*Data_Type_Dict.PM_SCALE['RX_PDL_SCALE_DB']

    PM_dict['rx_osnr_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_OSNR_PM)*Data_Type_Dict.PM_SCALE['OSNR_SCALE_DB']
    PM_dict['rx_osnr_min'] = read_reg_from_dict(port, Page35h.RX_MIN_OSNR_PM)*Data_Type_Dict.PM_SCALE['OSNR_SCALE_DB']
    PM_dict['rx_osnr_max'] = read_reg_from_dict(port, Page35h.RX_MAX_OSNR_PM)*Data_Type_Dict.PM_SCALE['OSNR_SCALE_DB']

    PM_dict['rx_esnr_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_ESNR_PM)*Data_Type_Dict.PM_SCALE['ESNR_SCALE_DB']
    PM_dict['rx_esnr_min'] = read_reg_from_dict(port, Page35h.RX_MIN_ESNR_PM)*Data_Type_Dict.PM_SCALE['ESNR_SCALE_DB']
    PM_dict['rx_esnr_max'] = read_reg_from_dict(port, Page35h.RX_MAX_ESNR_PM)*Data_Type_Dict.PM_SCALE['ESNR_SCALE_DB']

    PM_dict['rx_cfo_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_CFO_PM)
    PM_dict['rx_cfo_min'] = read_reg_from_dict(port, Page35h.RX_MIN_CFO_PM)
    PM_dict['rx_cfo_max'] = read_reg_from_dict(port, Page35h.RX_MAX_CFO_PM)

    PM_dict['rx_evm_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_EVM_PM)*Data_Type_Dict.PM_SCALE['EVM_SCALE']
    PM_dict['rx_evm_min'] = read_reg_from_dict(port, Page35h.RX_MIN_EVM_PM)*Data_Type_Dict.PM_SCALE['EVM_SCALE']
    PM_dict['rx_evm_max'] = read_reg_from_dict(port, Page35h.RX_MAX_EVM_PM)*Data_Type_Dict.PM_SCALE['EVM_SCALE']

    PM_dict['tx_power_avg'] = read_reg_from_dict(port, Page35h.TX_AVG_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']
    PM_dict['tx_power_min'] = read_reg_from_dict(port, Page35h.TX_MIN_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']
    PM_dict['tx_power_max'] = read_reg_from_dict(port, Page35h.TX_MAX_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']

    PM_dict['rx_power_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']
    PM_dict['rx_power_min'] = read_reg_from_dict(port, Page35h.RX_MIN_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']
    PM_dict['rx_power_max'] = read_reg_from_dict(port, Page35h.RX_MAX_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']

    PM_dict['rx_sigpwr_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_SIG_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']
    PM_dict['rx_sigpwr_min'] = read_reg_from_dict(port, Page35h.RX_MIN_SIG_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']
    PM_dict['rx_sigpwr_max'] = read_reg_from_dict(port, Page35h.RX_MAX_SIG_POWER_PM)*Data_Type_Dict.PM_SCALE['POWER_SCALE_DBM']

    PM_dict['rx_soproc_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_SIG_SOPROC_PM)
    PM_dict['rx_soproc_min'] = read_reg_from_dict(port, Page35h.RX_MIN_SIG_SOPROC_PM)
    PM_dict['rx_soproc_max'] = read_reg_from_dict(port, Page35h.RX_MAX_SIG_SOPROC_PM)

    PM_dict['rx_mer_avg'] = read_reg_from_dict(port, Page35h.RX_AVG_MER_PM)*Data_Type_Dict.PM_SCALE['MER_SCALE_DB']
    PM_dict['rx_mer_min'] = read_reg_from_dict(port, Page35h.RX_MIN_MER_PM)*Data_Type_Dict.PM_SCALE['MER_SCALE_DB']
    PM_dict['rx_mer_max'] = read_reg_from_dict(port, Page35h.RX_MAX_MER_PM)*Data_Type_Dict.PM_SCALE['MER_SCALE_DB']

    write_reg_from_dict(port, Page2Fh.FREEZE_REQUEST, 0)
    return PM_dict
```

#### Set module configuration, turn up
- set_low_power

```
def set_low_power(port, AssertLowPower):
    module_control = AssertLowPower << 6    
    write_reg_from_dict(port, Page00h_Lower.MODULE_LEVEL_CONTROL, module_control)
```
- set_TX_power

```
def set_TX_power(port, TX_power):
    
    # Target programmable output power increments of 0.01 dBm.
    TX_POWER_SCALE = 0.01
    TX_power_buffer = round(TX_power / TX_POWER_SCALE)
    write_reg_from_dict(port, Page12h.TX_TARGET_OUTPUT_POWER_LANE_1,TX_power_buffer)

    # Keep reading tuning in progress bit until it completes. TX_TUNING_STATUS_LANE_1
    # Bit 1: 0b, TX tuning not in progress; 1b, TX tuning in progress (both freq and power)
    # Bit 0: TX wavelength unlocked real-time status. 0b, wavelength locked; 1b, wavelength unlocked
    WAIT_TIME = 30 # wait for 30 seconds
    counter = 0
    while counter < WAIT_TIME:
        tuning_status = read_reg_from_dict(port, Page12h.TX_TUNING_STATUS_LANE_1) & 0x3
        if (tuning_status == 0):
            break
        time.sleep(1)
        counter += 1

    tuning_status = read_reg_from_dict(port, Page12h.TX_TUNING_STATUS_LANE_1) & 0x3
    if (tuning_status == 0):  # Tuning complete
        print('Success! Tuning completes!')
    else:
        print('Error! Tuning failed!')
```
- set_laser_freq

```
def set_laser_freq(port, freq):
    # Note the input argument freq must be in THz
    set_low_power(port, True)
    time.sleep(5)
    # Set selected grid spacing in Page 12h to 75 GHz
    # Bit 7-4: TX lane {n} selected grid spacing RW
    # Selected grid spacing of lane n=1-8.
    # 0000b: 3.125 GHz
    # 0001b: 6.25 GHz
    # 0010b: 12.5 GHz
    # 0011b: 25 GHz
    # 0100b: 50 GHz
    # 0101b: 100 GHz
    # 0110b: 33 GHz
    # 0111: 75 GHz
    # 8-14: Reserved
    # 1111b: Not available

    # Bit 0: TX lane {n} fine tuning enable RW
    # Bool fine-tuning enabled for lane n=1-8.
    # 0b: Fine-tuning disabled
    # 1b: Fine-tuning enabled.
    freq_grid = 0x70
    write_reg_from_dict(port, Page12h.TX_TUNING_SETTING_LANE_1, freq_grid)

    # Channel number n for 75 GHz grid spacing
    # Frequency (THz) = 193.1 + n * 0.025, where n must be divisible by 3.
    channel_number = int(round((freq - 193.1)/0.025))
    if channel_number % 3 is not 0:
        print('Error! Frequency has to be in 75 GHz grid!')
        return
    write_reg_from_dict(port, Page12h.TX_CHANNEL_NUM_LANE_1, channel_number)

    # Deassert low power mode
    set_low_power(port, False)
    
    # Keep reading tuning in progress bit until it completes. TX_TUNING_STATUS_LANE_1
    # Bit 1: 0b, TX tuning not in progress; 1b, TX tuning in progress (both freq and power)
    # Bit 0: TX wavelength unlocked real-time status. 0b, wavelength locked; 1b, wavelength unlocked
    WAIT_TIME = 30 # wait for 30 seconds
    counter = 0
    while counter < WAIT_TIME:
        tuning_status = read_reg_from_dict(port, Page12h.TX_TUNING_STATUS_LANE_1) & 0x3
        if (tuning_status == 0):
            break
        time.sleep(1)
        counter += 1

    tuning_status = read_reg_from_dict(port, Page12h.TX_TUNING_STATUS_LANE_1) & 0x3
    if (tuning_status == 0):  # Tuning complete
        print('Success! Tuning completes!')
    else:
        print('Error! Tuning failed!')
```
### Module firmware upgrade using command data block (CDB)
This section discusses the details of implementing firmware upgrade using CDB message communication. Figure 7-4 in [CMIS](http://www.qsfp-dd.com/wp-content/uploads/2019/05/QSFP-DD-CMIS-rev4p0.pdf) defines the flowchart for upgrading the module firmware. 

The first step is to obtain CDB features supported by the module from CDB command 0041h, such as start local payload size, maximum block size, whether extended payload messaging (page 0xA0 - 0xAF) or only local payload is supported. These features are important because the following upgrade with depend on these parameters. 

The second step is to start CDB download by writing the first 116 bytes (usually the header) from the designated firmware file to the local payload page 0x9F, with CDB command 0101h. 

The third step is to repeatedly read from the given firmware file and write to the payload space advertised from the first step. We use CDB command 0103h to write to the local payloadl; we use CDB command 0104h to write to the extended paylaod. This step repeats until it reaches end of the firmware file, or the CDB status failed.

The last step is to complete the firmware upgrade with CDB command 0107h. 

Note that if the download process fails anywhere in the middle, we need to run CDB command 0102h to abort the upgrade before we restart another upgrade process. 

After the firmware download finishes. The firmware will be run and committed with CDB command 0109h and 010Ah. We also check the currently running and committed firmware iamge version by CDB command 0100h before and after the entire firmware upgrade process to confirm the module switched to the updated firmware. 

A CDB command is triggered when byte 0x9F: 129 is written. This requires we break a CDB command into two written pieces:
1. write bytes from 0x9F: 130 till the message ends.
2. write bytes 0x9F: 128-129.
A single write command from 0x9F: 128 until the message ends should be avoided. 


Sample code to run firmware upgrade from high level functions:
```
import CDB_operations as cdb_oper

def module_firmware_upgrade(port, filepath):
    cdb_oper.get_module_FW_info(port)
    startLPLsize, maxblocksize, lplonly_flag = cdb_oper.get_module_FW_upgrade_features(port)
    cdb_oper.module_FW_download(port, startLPLsize, maxblocksize, lplonly_flag, filepath)
    cdb_oper.module_FW_run(port)
    time.sleep(60)
    cdb_oper.module_FW_commit(port)
    cdb_oper.get_module_FW_info(port)
```

Sample *CDB_operations* code (partial) to get module firmware information:
```
import cmisCDB as cdb

def get_module_FW_info(port):
    # get fw info (CMD 0100h)
    starttime = time.time()
    print('Get module FW info')
    rlplen, rlp_chkcode, rlp = cdb.cmd0100h(port)
    if cdb.cdb_chkcode(rlp) == rlp_chkcode:
        # Regiter 9Fh:136
        fwStatus = rlp[0]
        # Registers 9Fh:138,139; 140,141
        print('Image A Version: %d.%d; BuildNum: %d' %(rlp[2], rlp[3], ((rlp[4]<< 8) | rlp[5])))
        # Registers 9Fh:174,175; 176.177
        print('Image B Version: %d.%d; BuildNum: %d' %(rlp[38], rlp[39], ((rlp[40]<< 8) | rlp[41])))

        ImageARunning = (fwStatus & 0x01) # bit 0 - image A is running
        ImageACommitted = ((fwStatus >> 1) & 0x01) # bit 1 - image A is committed
        ImageBRunning = ((fwStatus >> 4) & 0x01) # bit 4 - image B is running
        ImageBCommitted = ((fwStatus >> 5) & 0x01)  # bit 5 - image B is committed

        if ImageARunning == 1: 
            RunningImage = 'A'
        elif ImageBRunning == 1:
            RunningImage = 'B'
        if ImageACommitted == 1:
            CommittedImage = 'A'
        elif ImageBCommitted == 1:
            CommittedImage = 'B'
        print('Running Image: %s; Committed Image: %s' %(RunningImage, CommittedImage))
    else:
        raise ValueError, 'Reply payload check code error'
    elapsedtime = time.time()-starttime
    print('Get module FW info time: %.2f s' %elapsedtime) 
```
The output of *get_module_FW_info* function:
```
>>> get_module_FW_info(port)
Get module FW info
Image A Version: 1.1; BuildNum: 4
Image B Version: 0.11; BuildNum: 127
Running Image: A; Committed Image: A
```

Sample code in *cmisCDB* to support *get_module_FW_info* function:

```
LPLPAGE = 0x9f
INIT_OFFSET = 128
CMDLEN = 2

# Get FW info
def cmd0100h(port):
    cmd = bytearray(b'\x01\x00\x00\x00\x00\x00\x00\x00')
    cmd[133-INIT_OFFSET] = cdb_chkcode(cmd)
    write_cdb(port,cmd)
    while cdb1_chkstatus(port):
        time.sleep(0.1)
    return read_cdb(port)
    
def cdb_chkcode(cmd):
    checksum = 0
    for byte in cmd:
        checksum += byte   
    return 0xff - (checksum & 0xff)

def cdb1_chkstatus(port):
    status = read_reg_from_dict(port, Page00h_Lower.CDB1_STATUS)
    return bool(status & 0x80)

def write_cdb(port,cmd):
    write_reg(port, LPLPAGE, INIT_OFFSET+CMDLEN, len(cmd)-CMDLEN, cmd[CMDLEN:])
    write_reg(port, LPLPAGE, INIT_OFFSET, CMDLEN, cmd[:CMDLEN])
```

