
# OAI and Commercial DU configuration comparison



| OAI Parameters                   | OAI value | TY value              | TY Parameters                    |
| -------------------------------- | --------- | --------------------- | -------------------------------- |
| tracking\_area\_code             | 1         | 0001                  | nRTAC                            |
| mcc                              | 001       | 001                   | mcc                              |
| mnc                              | 01        | 01                    | mnc                              |
| nr_cellid                        | 1         | 1                     | cellLocalId                      |
| **do_CSIRS**                       | 1         | false                 | isCsiRsEnable                    |
| **absoluteFrequencySSB**             | 630048    | 3708.480 MHz (647232) | absFreqSsb                       |
| dl_frequencyBand                 | 78        | 78                    | nrFreqBand                       |
| **dl_absoluteFrequencyPointA**       | 626772    | 3700.560 MHz (646704) | absFreqPointA                    |
| dl_carrierBandwidth              | 273       | 273                   | carrierBw                        |
| dl_subcarrierSpacing             | 1 (30kHz) | 1 (30kHz)             | subCarrierSpacing                |
| initialDLBWPlocationAndBandwidth | 1099      | 1099                  | initialDLBWPlocationAndBandwidth |
| pMax                             | 23        | 23                    | pMax                             |
| prach_ConfigurationIndex         | 159       | 159                   | prachCfgIdx                      |
| prach\_msg1\_FDM                 | 0 (=one)  | 1                     | msg1Fdm                          |
| prach\_msg1\_FrequencyStart      | 0         | 0                     | msg1FreqStart                    |
| **zeroCorrelationZoneConfig**        | 15        | 6                     | zeroCorrelationZoneCfg           |
| **preambleReceivedTargetPower**      | -104      | -84                   | preambleRcvdTgtPwr               |
| **preambleTransMax**                 | 7         | n10                   | preambleTransMax                 |
| **powerRampingStep**                 | 2 (dB4)   | dB2                   | pwrRampingStep                   |
| **ra_ResponseWindow**                | 5         | sl20                  | rachRspWindow                    |
| ssb\_PositionsInBurst\_PR        | 2         | 2                     | ssb\_PositionsInBurst\_PR        |
| ssb_periodicityServingCell       | 2 (ms20)  | MS20                  | ssbPeriodicity                   |
| dmrs\_TypeA\_Position            | 0         | 0                     | dmrs\_TypeA\_Position            |
| referenceSubcarrierSpacing       | 1         | 1                     | referenceSubcarrierSpacing       |
| dl\_UL\_TransmissionPeriodicity  | 5         | 5                     | dl\_UL\_TransmissionPeriodicity  |
| nrofDownlinkSlots                | 3         | 3                     | numDlSlot                        |
| **nrofDownlinkSymbols**              | 6         | 12                    | numDlSymbol                      |
| nrofUplinkSlots                  | 1         | 1                     | numUlSlot                        |
| **nrofUplinkSymbols**                | 4         | 0                     | numUlSymbol                      |


## RACH 
|                                       | OAI             | TY              |
| ------------------------------------------------ | --------------- | --------------- |
| **controlResourceSetZero**                       | 11              | 10              |
| **n_TimingAdvanceOffset**                        | Nan             | 1(25600)        |
| prach_ConfigurationIndex                         | 159             | 159             |
| prach_msg1_FDM                                   | 0               | 0               |
| prach_msg1_FrequencyStart                        | 0               | 0               |
| **zeroCorrelationZoneConfig**                    | 15              | 6               |
| **preambleReceivedTargetPower**                  | -104            | -84             |
| **preambleTransMax**                             | 7(20)           | 6(10)           |
| **powerRampingStep**                             | 2(dB4)          | 1(dB2)          |
| ra_ResponseWindow                                | 5(20)           | 5(20)           |
| **ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR** | 3(half)         | 4(one)          |
| **ssb_perRACH_OccasionAndCB_PreamblesPerSSB**    | 15(64)          | 1(8)            |
| **ra_ContentionResolutionTimer**                 | 7(64)           | 4(40)           |
| **rsrp_ThresholdSSB**                            | 19              | 31              |
| **prach_RootSequenceIndex**                      | 1               | 0               |
| msg1_SubcarrierSpacing                           | 1(30kHz)        | 1(30kHz)        |
| restrictedSetConfig                              | 0(unrestricted) | 0(unrestricted) |
| **msg3_DeltaPreamble**                           | 2               | 0               |
| **p0_NominalWithGrant**                          | -96             | -80             |
| pucchGroupHopping                                | 0(neither)      | 0(neither)      |
| **p0_nominal**                                   | -96             | -82             |