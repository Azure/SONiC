## CMIS and C-CMIS support for ZR on SONiC

### Overview
Common Management Interface Specification (CMIS) is defined for pluggables or on-board modules to communicate with the registers [CMIS v5.0](http://www.qsfp-dd.com/wp-content/uploads/2021/05/CMIS5p0.pdf). With a clear difinition of these registers, modules can set the configurations or get the status, to achieve the basic level of monitor and control. 

CMIS is widely used on modules based on a Two-Wire-Interface (TWI), including QSFP-DD, OSFP, COBO and QSFP modules. However, new requirements emerge with the introduction of coherent optical modules, such as 400G ZR. 400G ZR is the first type of modules to require definitions on coherent optical specifications, a field CMIS does not touch on. The development of C(coherent)-CMIS aims to solve this issue [C-CMIS v1.1](https://www.oiforum.com/wp-content/uploads/OIF-C-CMIS-01.1.pdf). It is based on CMIS but incroporates more definitions on registers in the extended space, regarding the emerging demands on coherent optics specifications.

The scope of this work is to develop APIs for both CMIS and C-CMIS to support 400G ZR modules on SONiC.

### State_DB and show transceiver CLI definitions:

#### State_DB Schema ####

New Transceiver info table and transceiver DOM sensor table adapted to 400G-ZR modules.

##### Transceiver info Table #####

    ; Defines Transceiver information for a port
    key                          = TRANSCEIVER_INFO|ifname          ; information for module on port
    ; field                      = value    
    type                         = 1*255VCHAR                       ; type of module
    module_media_type            = 1*255VCHAR                       ; module media interface ID
    host_electrical_interface    = 1*255VCHAR                       ; host electrical interface ID
    media_interface_code         = 1*255VCHAR                       ; media interface code
    host_lane_count              = 1*255VCHAR                       ; host lane count
    media_lane_count             = 1*255VCHAR                       ; media lane count
    host_lane_assigment_option   = 1*255VCHAR                       ; permissible first host lane number for application
    media_lane_assigment_option  = 1*255VCHAR                       ; permissible first media lane number for application
    active_apsel_hostlane1       = 1*255VCHAR                       ; active application selected code assigned to host lane 1
    active_apsel_hostlane2       = 1*255VCHAR                       ; active application selected code assigned to host lane 2
    active_apsel_hostlane3       = 1*255VCHAR                       ; active application selected code assigned to host lane 3
    active_apsel_hostlane4       = 1*255VCHAR                       ; active application selected code assigned to host lane 4
    active_apsel_hostlane5       = 1*255VCHAR                       ; active application selected code assigned to host lane 5
    active_apsel_hostlane6       = 1*255VCHAR                       ; active application selected code assigned to host lane 6
    active_apsel_hostlane7       = 1*255VCHAR                       ; active application selected code assigned to host lane 7
    active_apsel_hostlane8       = 1*255VCHAR                       ; active application selected code assigned to host lane 8
    media_interface_technology   = 1*255VCHAR                       ; media interface technology
    hardwarerev                  = 1*255VCHAR                       ; module hardware revision 
    serialnum                    = 1*255VCHAR                       ; module serial number 
    manufacturename              = 1*255VCHAR                       ; module venndor name
    modelname                    = 1*255VCHAR                       ; module model name
    vendor_oui                   = 1*255VCHAR                       ; vendor organizationally unique identifier
    vendor_date                  = 1*255VCHAR                       ; module manufacture date
    Connector                    = 1*255VCHAR                       ; connector type
    encoding                     = 1*255VCHAR                       ; serial encoding mechanism
    ext_identifier               = 1*255VCHAR                       ; additional infomation about the sfp
    ext_rateselect_compliance    = 1*255VCHAR                       ; additional rate select compliance information
    cable_type                   = 1*255VCHAR                       ; cable type
    cable_length                 = 1*255VCHAR                       ; cable length that supported
    specification_compliance     = 1*255VCHAR                       ; electronic or optical interfaces that supported
    nominal_bit_rate             = 1*255VCHAR                       ; nominal bit rate per channel
    firmware_major_rev           = 1*255VCHAR                       ; firmware major revision
    firmware_minor_rev           = 1*255VCHAR                       ; firmware minor revision


##### Transceiver DOM sensor Table #####

    ; Defines Transceiver DOM sensor information for a port
    key                          = TRANSCEIVER_DOM_SENSOR|ifname    ; information module DOM sensors on port
    temperature                  = FLOAT                            ; temperature value in Celsius
    voltage                      = FLOAT                            ; voltage value
    rx1power                     = FLOAT                            ; rx1 power in dbm
    rx2power                     = FLOAT                            ; rx2 power in dbm
    rx3power                     = FLOAT                            ; rx3 power in dbm
    rx4power                     = FLOAT                            ; rx4 power in dbm
    tx1bias                      = FLOAT                            ; tx1 bias in mA
    tx2bias                      = FLOAT                            ; tx2 bias in mA
    tx3bias                      = FLOAT                            ; tx3 bias in mA
    tx4bias                      = FLOAT                            ; tx4 bias in mA
    laser_temperature	         = FLOAT                            ; laser temperature value in Celsius
    prefec_ber                   = FLOAT                            ; prefec ber
    postfec_ber                  = FLOAT                            ; postfec ber
    cd_shortlink                 = FLOAT                            ; chromatic dispersion, high granularity, short link in ps/nm
    cd_longlink                  = FLOAT                            ; chromatic dispersion, low granularity, long link in ps/nm
    dgd                          = FLOAT                            ; differential group delay in ps
    sopmd                        = FLOAT                            ; second order polarization mode dispersion in ps^2
    pdl                          = FLOAT                            ; polarization dependent loss in db
    osnr                         = FLOAT                            ; optical signal to noise ratio in db
    esnr                         = FLOAT                            ; electrical signal to noise ratio in db
    cfo                          = FLOAT                            ; carrier frequency offset in MHz
    soproc                       = FLOAT                            ; state of polarization rate of change in krad/s
    laser_config_freq            = FLOAT                            ; laser configured frequency in MHz
    laser_curr_freq              = FLOAT                            ; laser current frequency in MHz
    tx_config_power              = FLOAT                            ; configured tx output power in dbm
    tx_curr_power                = FLOAT                            ; tx current output power in dbm
    rx_tot_power                 = FLOAT                            ; rx total power in  dbm
    rx_sig_power                 = FLOAT                            ; rx signal power in dbm
    media_output_loopback        = FLOAT                            ; media side output loopback enable
    media_input_loopback         = FLOAT                            ; media side input loopback enable
    host_output_loopback_lane1   = FLOAT                            ; host side output loopback enable lane1
    host_output_loopback_lane2   = FLOAT                            ; host side output loopback enable lane2
    host_output_loopback_lane3   = FLOAT                            ; host side output loopback enable lane3
    host_output_loopback_lane4   = FLOAT                            ; host side output loopback enable lane4
    host_output_loopback_lane5   = FLOAT                            ; host side output loopback enable lane5
    host_output_loopback_lane6   = FLOAT                            ; host side output loopback enable lane6
    host_output_loopback_lane7   = FLOAT                            ; host side output loopback enable lane7
    host_output_loopback_lane8   = FLOAT                            ; host side output loopback enable lane8
    host_intput_loopback_lane1   = FLOAT                            ; host side intput loopback enable lane1
    host_intput_loopback_lane2   = FLOAT                            ; host side intput loopback enable lane2
    host_intput_loopback_lane3   = FLOAT                            ; host side intput loopback enable lane3
    host_intput_loopback_lane4   = FLOAT                            ; host side intput loopback enable lane4
    host_intput_loopback_lane5   = FLOAT                            ; host side intput loopback enable lane5
    host_intput_loopback_lane6   = FLOAT                            ; host side intput loopback enable lane6
    host_intput_loopback_lane7   = FLOAT                            ; host side intput loopback enable lane7
    host_intput_loopback_lane8   = FLOAT                            ; host side intput loopback enable lane8

##### Transceiver Status Table #####

    ; Defines Transceiver Status info for a port
    key                          = TRANSCEIVER_STATUS|ifname        ; Error information for module on port
    ; field                      = value    
    status                       = 1*255VCHAR                       ; code of the module status (plug in, plug out)
    error                        = 1*255VCHAR                       ; module error (N/A or a string consisting of error descriptions joined by "|", like "error1 | error2" )
    module_state                 = 1*255VCHAR                       ; current module state (ModuleLowPwr, ModulePwrUp, ModuleReady, ModulePwrDn, Fault)
    module_fault_cause           = 1*255VCHAR                       ; reason of entering the module fault state
    datapath_firmware_fault      = 1*255VCHAR                       ; datapath (DSP) firmware fault
    module_firmware_fault        = 1*255VCHAR                       ; module firmware fault
    module_state_changed         = 1*255VCHAR                       ; module state changed
    datapath_hostlane1           = 1*255VCHAR                       ; data path state indicator on host lane 1
    datapath_hostlane2           = 1*255VCHAR                       ; data path state indicator on host lane 2
    datapath_hostlane3           = 1*255VCHAR                       ; data path state indicator on host lane 3
    datapath_hostlane4           = 1*255VCHAR                       ; data path state indicator on host lane 4
    datapath_hostlane5           = 1*255VCHAR                       ; data path state indicator on host lane 5
    datapath_hostlane6           = 1*255VCHAR                       ; data path state indicator on host lane 6
    datapath_hostlane7           = 1*255VCHAR                       ; data path state indicator on host lane 7
    datapath_hostlane8           = 1*255VCHAR                       ; data path state indicator on host lane 8
    txoutput_status              = 1*255VCHAR                       ; tx output status
    rxoutput_status              = 1*255VCHAR                       ; rx output status
    txfault                      = 1*255VCHAR                       ; tx fault flag
    txlos                        = 1*255VCHAR                       ; tx loss of signal flag
    txcdrlol                     = 1*255VCHAR                       ; tx clock and data recovery loss of lock
    rxlos                        = 1*255VCHAR                       ; rx loss of signal flag
    rxcdrlol                     = 1*255VCHAR                       ; rx clock and data recovery loss of lock
    config_state_hostlane1       = 1*255VCHAR                       ; configuration status for the data path of host line 1
    config_state_hostlane2       = 1*255VCHAR                       ; configuration status for the data path of host line 2
    config_state_hostlane3       = 1*255VCHAR                       ; configuration status for the data path of host line 3
    config_state_hostlane4       = 1*255VCHAR                       ; configuration status for the data path of host line 4
    config_state_hostlane5       = 1*255VCHAR                       ; configuration status for the data path of host line 5
    config_state_hostlane6       = 1*255VCHAR                       ; configuration status for the data path of host line 6
    config_state_hostlane7       = 1*255VCHAR                       ; configuration status for the data path of host line 7
    config_state_hostlane8       = 1*255VCHAR                       ; configuration status for the data path of host line 8
    dpinit_pending_hostlane1     = 1*255VCHAR                       ; data path configuration updated on host lane 1 
    dpinit_pending_hostlane2     = 1*255VCHAR                       ; data path configuration updated on host lane 2
    dpinit_pending_hostlane3     = 1*255VCHAR                       ; data path configuration updated on host lane 3
    dpinit_pending_hostlane4     = 1*255VCHAR                       ; data path configuration updated on host lane 4
    dpinit_pending_hostlane5     = 1*255VCHAR                       ; data path configuration updated on host lane 5
    dpinit_pending_hostlane6     = 1*255VCHAR                       ; data path configuration updated on host lane 6
    dpinit_pending_hostlane7     = 1*255VCHAR                       ; data path configuration updated on host lane 7
    dpinit_pending_hostlane8     = 1*255VCHAR                       ; data path configuration updated on host lane 8
    tuning_in_progress           = 1*255VCHAR                       ; tuning in progress status
    wavelength_unlock_status     = 1*255VCHAR                       ; laser unlocked status
    target_output_power_oor      = 1*255VCHAR                       ; target output power out of range flag
    fine_tuning_oor              = 1*255VCHAR                       ; fine tuning  out of range flag
    tuning_not_accepted          = 1*255VCHAR                       ; tuning not accepted flag
    invalid_channel_num          = 1*255VCHAR                       ; invalid channel number flag
    tuning_complete              = 1*255VCHAR                       ; tuning complete flag
    temphighalarm                = 1*255VCHAR                       ; temperature high alarm threshold 
    temphighwarning              = 1*255VCHAR                       ; temperature high warning threshold
    templowalarm                 = 1*255VCHAR                       ; temperature low alarm threshold
    templowwarning               = 1*255VCHAR                       ; temperature low warning threshold
    vcchighalarm                 = 1*255VCHAR                       ; vcc high alarm threshold
    vcchighwarning               = 1*255VCHAR                       ; vcc high warning threshold
    vcclowalarm                  = 1*255VCHAR                       ; vcc low alarm threshold
    vcclowwarning                = 1*255VCHAR                       ; vcc low warning threshold
    txpowerhighalarm             = 1*255VCHAR                       ; tx power high alarm threshold
    txpowerlowalarm              = 1*255VCHAR                       ; tx power low alarm threshold
    txpowerhighwarning           = 1*255VCHAR                       ; tx power high warning threshold
    txpowerlowwarning            = 1*255VCHAR                       ; tx power low alarm threshold
    rxpowerhighalarm             = 1*255VCHAR                       ; rx power high alarm threshold
    rxpowerlowalarm              = 1*255VCHAR                       ; rx power low alarm threshold
    rxpowerhighwarning           = 1*255VCHAR                       ; rx power high warning threshold
    rxpowerlowwarning            = 1*255VCHAR                       ; rx power low warning threshold
    txbiashighalarm              = 1*255VCHAR                       ; tx bias high alarm threshold
    txbiaslowalarm               = 1*255VCHAR                       ; tx bias low alarm threshold
    txbiashighwarning            = 1*255VCHAR                       ; tx bias high warning threshold
    txbiaslowwarning             = 1*255VCHAR                       ; tx bias low warning threshold
    lasertemphighalarm           = 1*255VCHAR                       ; laser temperature high alarm threshold
    lasertemplowalarm            = 1*255VCHAR                       ; laser temperature low alarm threshold
    lasertemphighwarning         = 1*255VCHAR                       ; laser temperature high warning threshold
    lasertemplowwarning          = 1*255VCHAR                       ; laser temperature low warning threshold
    
##### Transceiver PM Table #####

    ; Defines Transceiver PM information for a port
    key                          = TRANSCEIVER_PM|ifname            ; information of PM on port
    prefec_ber_avg               = FLOAT                            ; prefec ber avg
    prefec_ber_min               = FLOAT                            ; prefec ber min
    prefec_ber_max               = FLOAT                            ; prefec ber max
    uncorr_frames_avg            = FLOAT                            ; uncorrected frames ratio avg
    uncorr_frames_min            = FLOAT                            ; uncorrected frames ratio min
    uncorr_frames_max            = FLOAT                            ; uncorrected frames ratio max
    cd_avg                       = FLOAT                            ; chromatic dispersion avg
    cd_min                       = FLOAT                            ; chromatic dispersion min
    cd_max                       = FLOAT                            ; chromatic dispersion max
    dgd_avg                      = FLOAT                            ; differential group delay avg
    dgd_min                      = FLOAT                            ; differential group delay min
    dgd_max                      = FLOAT                            ; differential group delay max
    sopmd_avg                    = FLOAT                            ; second order polarization mode dispersion avg
    sopmd_min                    = FLOAT                            ; second order polarization mode dispersion min
    sopmd_max                    = FLOAT                            ; second order polarization mode dispersion max
    pdl_avg                      = FLOAT                            ; polarization dependent loss avg
    pdl_min                      = FLOAT                            ; polarization dependent loss min
    pdl_max                      = FLOAT                            ; polarization dependent loss max
    osnr_avg                     = FLOAT                            ; optical signal to noise ratio avg
    osnr_min                     = FLOAT                            ; optical signal to noise ratio min
    osnr_max                     = FLOAT                            ; optical signal to noise ratio max
    esnr_avg                     = FLOAT                            ; electrical signal to noise ratio avg
    esnr_min                     = FLOAT                            ; electrical signal to noise ratio min
    esnr_max                     = FLOAT                            ; electrical signal to noise ratio max
    cfo_avg                      = FLOAT                            ; carrier frequency offset avg
    cfo_min                      = FLOAT                            ; carrier frequency offset min
    cfo_max                      = FLOAT                            ; carrier frequency offset max
    soproc_avg                   = FLOAT                            ; state of polarization rate of change avg
    soproc_min                   = FLOAT                            ; state of polarization rate of change min
    soproc_max                   = FLOAT                            ; state of polarization rate of change max
    tx_power_avg                 = FLOAT                            ; tx output power avg
    tx_power_min                 = FLOAT                            ; tx output power min
    tx_power_max                 = FLOAT                            ; tx output power max
    rx_tot_power_avg             = FLOAT                            ; rx total power avg
    rx_tot_power_min             = FLOAT                            ; rx total power min
    rx_tot_power_max             = FLOAT                            ; rx total power max
    rx_sig_power_avg             = FLOAT                            ; rx signal power avg
    rx_sig_power_min             = FLOAT                            ; rx signal power min
    rx_sig_power_max             = FLOAT                            ; rx signal power max 
    
#### Show interfaces transceiver CLI
Displays diagnostic monitoring information of the transceivers

**show interfaces transceiver**

This command displays information for all the interfaces for the transceiver requested or a specific interface if the optional "interface_name" is specified.

- Usage:
  ```
  show interfaces transceiver (info | eeprom [-d|--dom] | presence | status | pm) [<interface_name>]
  ```
- Example (Decode and display general information of the transceiver connected to Ethernet0):
  ```
  admin@sonic:~$ show interfaces transceiver info Ethernet0
  Ethernet0:
  Module Type :  QSFP-DD Double Density 8X Pluggable Transceiver (INF-8628)
  Media Type : Single Model Fiber (SMF)
  Host Electrical Interface : 400GAUI-8 C2M (Annex 120E), data rate 425.00, lane count 8, lane signal baudrate 26.5625, modulation PAM4, bit per symbol 2
  Media Interface Code : TBD
  Host Lane Count : 8
  Media Lane Count : 1
  Host Lane Assignment Options : 1
  Media Lane Assignment Options : 1
  Active App Selection Host Lane 1 : 1
  Active App Selection Host Lane 2 : 1
  Active App Selection Host Lane 3 : 1
  Active App Selection Host Lane 4 : 1
  Active App Selection Host Lane 5 : 1
  Active App Selection Host Lane 6 : 1
  Active App Selection Host Lane 7 : 1
  Active App Selection Host Lane 8 : 1
  Media Interface Technology : C-band tunable laser
  Vendor Name : XXXXXXX
  Vendor OUI : XX-XX-XX
  Hardware Revision : XX.XX
  Module PN : XXXXXXXXXXX
  Module SN : XXXXXXXXXXX
  Module Manufacture Date : XXXXXXXX (MMDDYYYY)
  Connector Type : LC (Lucent Connector)
  CMIS Revision : X.X
  Firmware Version :  XXX.XXX
  ```

- Example (Decode and display dom information of the transceiver connected to Ethernet0):
  ```
  admin@sonic:~$ show interfaces transceiver eeprom --dom Ethernet0
  Ethernet0:
  Temperature : 45.00C
  Vcc : 3.33Volts
  TX1Bias : 70mA
  Laser Temperature : 30C
  Prefec Ber : 1.00E-3
  Postfec Ber : 0.00E0
  Cd Shortlink : 2000 ps/ns
  Cd Longlink : 2000 ps/ns
  Dgd : 1.0 ps
  Sopmd : 1 ps^2
  Pdl : 1.0 dB
  Osnr : 28.00 dB
  Esnr : 15.00 dB
  Cfo : 500 MHz
  Soproc : 1 krad/s
  Laser Config Frequency: 193100000 MHz
  Laser Current Frequency : 193100000 MHz
  Tx Config Power : -10.00 dBm
  Tx Current Power : -10.00 dBm
  Rx Total Power : -8.00 dBm
  Rx Signal Power : -8.00 dBm
  Media Output Loopback : False
  Media Input Loopback : False
  Host Output Loopback : False
  Host Input Loopback : False
  ```

- Example (Display presence of SFP transceiver connected to Ethernet0):
  ```
  admin@sonic:~$ show interfaces transceiver presence Ethernet0
  Port         Low-power Mode
  -----------  ----------------
  Ethernet0  On
  ```

- Example (Display status of transceiver connected to Ethernet0):
  ```
  admin@sonic:~$ show interfaces transceiver status Ethernet0
  Ethernet0:
  Module State : Ready
  Module Fault Cause : No Fault detected
  Datapath Firmware Fault : False
  Module Firmware Fault : False
  Module State Changed : False
  Datapath Host Lane 1: Activated
  Datapath Host Lane 2: Activated
  Datapath Host Lane 3: Activated
  Datapath Host Lane 4: Activated
  Datapath Host Lane 5: Activated
  Datapath Host Lane 6: Activated
  Datapath Host Lane 7: Activated
  Datapath Host Lane 8: Activated
  Tx Output Status : Valid
  Rx Output Status : Valid  
  Tx Fault : False
  Tx Los : False
  Tx Cdr Lol: False
  Rx Los : False
  Rx Cdr Lol : False
  Configuration State Host Lane 1: Config Success
  Configuration State Host Lane 2: Config Success
  Configuration State Host Lane 3: Config Success
  Configuration State Host Lane 4: Config Success
  Configuration State Host Lane 5: Config Success
  Configuration State Host Lane 6: Config Success
  Configuration State Host Lane 7: Config Success
  Configuration State Host Lane 8: Config Success
  Datapath Init Pending Host Lane 1: DPInit not pending
  Datapath Init Pending Host Lane 2: DPInit not pending
  Datapath Init Pending Host Lane 3: DPInit not pending
  Datapath Init Pending Host Lane 4: DPInit not pending
  Datapath Init Pending Host Lane 5: DPInit not pending
  Datapath Init Pending Host Lane 6: DPInit not pending
  Datapath Init Pending Host Lane 7: DPInit not pending
  Datapath Init Pending Host Lane 8: DPInit not pending
  Tuning In Progress: Tuning Not In Progress
  Wavelegnth Unlock Status : Wavelength Locked
  Target Output Power Out Of Range : False
  Fine Tuning Out Of Range : False
  Tuning Not Accepted : False
  Invalid Channel Number : False
  Tuning Complete : True
  Temperature High Alarm : False
  Temperature High Warning : False
  Temperature Low Alarm : False
  Temperature Low Warning : False
  Vcc High Alarm : False
  Vcc High Warning : False
  Vcc Low Alarm : False
  Vcc Low Warning : False
  Tx Power High Alarm : False
  Tx Power High Warning : False
  Tx Power Low Alarm : False
  Tx Power Low Warning : False
  Rx Power High Alarm : False
  Rx Power High Warning : False
  Rx Power Low Alarm : False
  Rx Power Low Warning : False  
  Tx Bias High Alarm : False
  Tx Bias High Warning : False
  Tx Bias Low Alarm : False
  Tx Bias Low Warning : False  
  Laser Temperature High Alarm : False
  Laser Temperature High Warning : False
  Laser Temperature Low Alarm : False
  Laser Temperature Low Warning : False
  ```
- Example (Display pm status of transceiver connected to Ethernet0):
  ```
  admin@sonic:~$ show interfaces transceiver pm Ethernet0
  Ethernet0:
  prefec_ber_avg : 1.00E-3
  prefec_ber_min : 1.00E-3
  prefec_ber_max : 1.00E-3
  uncorr_frames_avg : 0.00E0
  uncorr_frames_min : 0.00E0
  uncorr_frames_max : 0.00E0
  cd_avg : 0 ps/ns
  cd_min : 0 ps/ns
  cd_max : 0 ps/ns
  dgd_avg : 1.0 ps
  dgd_min : 1.0 ps
  dgd_max : 1.0 ps
  sopmd_avg : 1 ps^2
  sopmd_min : 1 ps^2
  sopmd_max : 1 ps^2
  pdl_avg  : 1.0 dB
  pdl_min  : 1.0 dB
  pdl_max  : 1.0 dB
  osnr_avg : 28.0 dB
  osnr_min : 28.0 dB
  osnr_max : 28.0 dB
  esnr_avg : 15.0 dB
  esnr_min : 15.0 dB
  esnr_max : 15.0 dB
  cfo_avg  : 500 MHz
  cfo_min  : 500 MHz
  cfo_max  : 500 MHz
  soproc_avg : 1 krad/s
  soproc_min : 1 krad/s
  soproc_max : 1 krad/s
  tx_power_avg : -10.00 dBm
  tx_power_min : -10.00 dBm
  tx_power_max : -10.00 dBm
  rx_tot_power_avg : -8.00 dBm
  rx_tot_power_min : -8.00 dBm
  rx_tot_power_max : -8.00 dBm
  rx_sig_power_avg : -8.00 dBm
  rx_sig_power_min : -8.00 dBm
  rx_sig_power_max : -8.00 dBm
  ```
  
#### Config_DB Schema ####
##### Port Table #####
Stores information for physical switch ports managed by the switch chip. Ports to the CPU (ie: management port) and logical ports (loopback) are not declared in the PORT_TABLE. See INTF_TABLE.

    ;Defines layer 2 ports
    ;In SONiC, Data is loaded from configuration file by portsyncd
    key                 = PORT_TABLE:ifname     ; ifname must be unique across PORT,INTF,VLAN,LAG TABLES
    admin_status        = "down" / "up"         ; admin status
    oper_status         = "down" / "up"         ; oper status
    lanes               = list of lanes         ; (need format spec???)
    mac                 = 12HEXDIG              ;
    alias               = 1*64VCHAR             ; alias name of the port used by LLDP and SNMP, must be unique
    description         = 1*64VCHAR             ; port description
    speed               = 1*6DIGIT              ; port line speed in Mbps
    mtu                 = 1*4DIGIT              ; port MTU
    fec                 = 1*64VCHAR             ; port fec mode
    lpmode              = "enable" / "disable"  ; port low power mode
    configured_freq     = 1*9DIGIT              ; configured frequency
    configured_TX_power = FLOAT                 ; configured TX output power 
    loopback            = "media output" / "media input" / "host output" / "host input" / "none" ; loopback mode
    
    
#### configure interfaces transceiver CLI
configure privisioning settings of the transceivers

- Usage:
    ```
    configure interfaces transceiver [<interface_name>] (lpmode | configured_frequency | configured_tx_power | loopback) 
    ```

- Example (bring module up from low power mode, or bring down module to low power mode):
    ```
    admin@sonic:~$ configure interfaces transceiver Ethernet0 lpmode disable
    ```

- Example (configure the privisioning frequency):
    ```
    admin@sonic:~$ configure interfaces transceiver Ethernet0 configured_frequency 193100000
    ```

- Example (configure the privisioning TX power):
    ```
    admin@sonic:~$ configure interfaces transceiver Ethernet0 configured_tx_power -10.00
    ```

- Example (configure the loopback mode):
    ```
    admin@sonic:~$ configure interfaces transceiver Ethernet0 loopback none
    ```    
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

-  Module general information pages (Page 0h - 1Fh, CMIS). See Figure 8-1 and Figure 8-2 in [CMIS](http://www.qsfp-dd.com/wp-content/uploads/2021/05/CMIS5p0.pdf)

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

VDM Pages. See Table 8-119 in [CMIS](http://www.qsfp-dd.com/wp-content/uploads/2021/05/CMIS5p0.pdf)

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
This section discusses the details of implementing firmware upgrade using CDB message communication. Figure 7-4 in [CMIS](http://www.qsfp-dd.com/wp-content/uploads/2021/05/CMIS5p0.pdf) defines the flowchart for upgrading the module firmware. 

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

