# ZX’s Random access MSC

<aside>

Reference:

[](https://hackmd.io/8sYry3iQSSWo3Ve13lVK1w?view#List-log-details-on-MSC-and-explain)

</aside>

![image.png](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/image.png)

## MSG1

Related to Msg1 receive and process log

```bash
[0m[NR_PHY]   [RAPROC] 695.19 Initiating RA procedure with preamble 49, energy 56.4 dB (I0 154, thres 120), delay 0 start symbol 0 freq index 0
[0m[NR_MAC]   695.19 UE RA-RNTI 010b TC-RNTI 7e5f: Activating RA process index 0
```

![image.png](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/image%201.png)

OAI source code

```c
void L1_nr_prach_procedures(PHY_VARS_gNB *gNB,int frame,int slot) {
....
    if ((gNB->prach_energy_counter == 100) && (max_preamble_energy[0] > gNB->measurements.prach_I0 + gNB->prach_thres)) {
        LOG_I(NR_PHY,
              "[RAPROC] %d.%d Initiating RA procedure with preamble %d, energy %d.%d dB (I0 %d, thres %d), delay %d start symbol "
              "%u freq index %u\n",
              frame,
              slot,
              max_preamble[0],
              max_preamble_energy[0] / 10,
              max_preamble_energy[0] % 10,
              gNB->measurements.prach_I0,
              gNB->prach_thres,
              max_preamble_delay[0],
              prachStartSymbol,
              prach_pdu->num_ra);
....
```

`rx_nr_prach` calculates **energy** and **delay**

```c
void rx_nr_prach(PHY_VARS_gNB *gNB,
                 nfapi_nr_prach_pdu_t *prach_pdu,
                 int prachOccasion,
                 int frame,
                 int slot,
                 uint16_t *max_preamble,
                 uint16_t *max_preamble_energy,
                 uint16_t *max_preamble_delay)
{
....
    for (i=0; i<NCS2; i++) {
      lev = (int32_t)prach_ifft[(preamble_shift2+i)];
      levdB = dB_fixed_times10(lev);// signal dB calculation
      if (levdB>*max_preamble_energy) {
        LOG_D(PHY,"preamble_index %d, delay %d en %d dB > %d dB\n",preamble_index,i,levdB,*max_preamble_energy);
        *max_preamble_energy  = levdB;
        *max_preamble_delay   = i; // Note: This has to be normalized to the 30.72 Ms/s sampling rate
        *max_preamble         = preamble_index;
      }
    }
.....
```

> Main PRACH Detection Loop:
> 
> 
> ![image.png](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/image%202.png)
> 
> 1. **Iterate over preambles** (up to 64):
>     - The code calculates a **preamble offset** based on `NCS` and checks if a new DFT (Discrete Fourier Transform) is required.
>     - If the preamble is in a **restricted set**, additional logic calculates cyclic shifts for high-speed scenarios and manages sequence offsets.
> 2. **DFT and Energy Computation**:
>     - For new DFTs, the function computes the DFT on the received signal `rxsigF` and correlates it with the PRACH root sequence.
>     - It performs an IFFT (Inverse Fast Fourier Transform) to transform data to the time domain.
>     - The energy of the transformed signal is computed and accumulated across receive antennas.
> 3. **Energy Check**:
>     - The code scans for the energy in different time shifts within the IFFT output to find the highest peak (indicating the most likely preamble).
>     - If a new maximum energy is found, it updates `max_preamble`, `max_preamble_energy`, and `max_preamble_delay`.
> 
> ### Timing Advance Adjustment:
> 
> - The detected delay (`max_preamble_delay`) is adjusted to align with the 30.72 Ms/s sampling rate based on the PRACH format and numerology (`mu`).
> 
> ### Utility Functions and Operations:
> 
> - **`signal_energy`**: Computes energy from a given signal.
> - **`dB_fixed_times10`**: Converts an energy value to decibels scaled by 10.
> - **`idft`**: Performs the Inverse Discrete Fourier Transform to bring signal data to the time domain.
> 
> ### Important Logic Points:
> 
> - **New DFTs** are only computed when necessary (when `preamble_offset` changes).
> - **Normalization** of IFFT output and accumulated energy is based on the number of receive antennas and the size of the IFFT.
> 
> ### Final Considerations:
> 
> The function processes incoming PRACH signals, correlates them with possible preamble sequences, and determines the best match based on energy, yielding the preamble index, energy level, and timing delay for transmission handling.
> 

```c
void nr_initiate_ra_proc(module_id_t module_idP,
                         int CC_id,
                         frame_t frameP,
                         sub_frame_t slotP,
                         uint16_t preamble_index,
                         uint8_t freq_index,
                         uint8_t symbol,
                         int16_t timing_offset)
{
  VCD_SIGNAL_DUMPER_DUMP_FUNCTION_BY_NAME(VCD_SIGNAL_DUMPER_FUNCTIONS_INITIATE_RA_PROC, 1);

  gNB_MAC_INST *nr_mac = RC.nrmac[module_idP];
  NR_SCHED_LOCK(&nr_mac->sched_lock);

  NR_COMMON_channels_t *cc = &nr_mac->common_channels[CC_id];

  NR_RA_t *ra = find_free_nr_RA(cc->ra, sizeofArray(cc->ra), preamble_index);
  if (ra == NULL) {
    LOG_E(NR_MAC, "FAILURE: %4d.%2d initiating RA procedure for preamble index %d: no free RA process\n", frameP, slotP, preamble_index);
    NR_SCHED_UNLOCK(&nr_mac->sched_lock);
    VCD_SIGNAL_DUMPER_DUMP_FUNCTION_BY_NAME(VCD_SIGNAL_DUMPER_FUNCTIONS_INITIATE_RA_PROC, 0);
    return;
  }

  if (ra->rnti == 0) { // This condition allows for the usage of a preconfigured rnti for the CFRA
    int loop = 0;
    bool exist_connected_ue, exist_in_pending_ra_ue;
    rnti_t trial = 0;
    do {
      // 3GPP TS 38.321 version 15.13.0 Section 7.1 Table 7.1-1: RNTI values
      while (trial < 1 || trial > 0xffef)
        trial = (taus() % 0xffef) + 1;
      exist_connected_ue = find_nr_UE(&nr_mac->UE_info, trial) != NULL;
      exist_in_pending_ra_ue = find_nr_RA_rnti(cc->ra, sizeofArray(cc->ra), ra->rnti) != NULL;
      loop++;
    } while (loop < 100 && (exist_connected_ue || exist_in_pending_ra_ue) );
    if (loop == 100) {
      LOG_E(NR_MAC, "initialisation random access: no more available RNTIs for new UE\n");
      NR_SCHED_UNLOCK(&nr_mac->sched_lock);
      return;
    }
    ra->rnti = trial;
  }

  ra->ra_state = nrRA_Msg2;
  ra->preamble_frame = frameP;
  ra->preamble_slot = slotP;
  ra->preamble_index = preamble_index;
  ra->timing_offset = timing_offset;
  uint8_t ul_carrier_id = 0; // 0 for NUL 1 for SUL
  ra->RA_rnti = nr_get_ra_rnti(symbol, slotP, freq_index, ul_carrier_id);
  int index = ra - cc->ra;
  LOG_I(NR_MAC, "%d.%d UE RA-RNTI %04x TC-RNTI %04x: Activating RA process index %d\n", frameP, slotP, ra->RA_rnti, ra->rnti, index);

  // Configure RA BWP
  NR_ServingCellConfigCommon_t *scc = cc->ServingCellConfigCommon;
  configure_UE_BWP(nr_mac, scc, NULL, ra, NULL, -1, -1);

  uint8_t beam_index = ssb_index_from_prach(module_idP, frameP, slotP, preamble_index, freq_index, symbol);
  ra->beam_id = cc->ssb_index[beam_index];

  NR_SCHED_UNLOCK(&nr_mac->sched_lock);

  VCD_SIGNAL_DUMPER_DUMP_FUNCTION_BY_NAME(VCD_SIGNAL_DUMPER_FUNCTIONS_INITIATE_RA_PROC, 0);
}
```

> 
> 
> 
> ### Key Steps
> 
> 1. **Locks Scheduling**: `NR_SCHED_LOCK` and `NR_SCHED_UNLOCK` protect shared resources from concurrent access.
> 2. **RA Process Search**: Finds a free RA process using `find_free_nr_RA()`. If none are available, an error is logged, and the function exits.
> 3. **RNTI Assignment**:
>     - If the RA structure has an unassigned `rnti` (Radio Network Temporary Identifier), it enters a loop to assign a unique RNTI.(TC-RNTI)
>     - It checks for RNTI **uniqueness** among connected UEs and pending RA processes.
> 4. **RA State**: Sets the RA state to `nrRA_Msg2`, indicating that the RA procedure is at the second step (Msg2).
> 5. **RA RNTI Generation**: Computes the RA-RNTI, which uniquely identifies the RA process during communication.
> 6. **Logging**: Logs important information like RA-RNTI, TC-RNTI, and RA process index for monitoring.
> 7. **BWP Configuration**: Configures the Bandwidth Part (BWP) for the UE using `configure_UE_BWP()`.
> 8. **Beam ID Assignment**: Determines the appropriate beam index for communication based on the PRACH (Physical Random Access Channel) configuration.
> 
> ### Output
> 
> The function does not return a value but logs information and modifies the RA state in the MAC instance. If errors occur (e.g., no available RNTIs or RA processes), appropriate logs are generated and the function exits early.
> 

![image.png](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/image%203.png)

> According to 3GPP TS 38.321 7.1 Table 7.1-1 中的 RNTI 值，合法的 RNTI 範圍是從 1 到 0xffef（65519）。
> 

```c
// RA-RNTI computation (associated to PRACH occasion in which the RA Preamble is transmitted)
// - this does not apply to contention-free RA Preamble for beam failure recovery request
// - getting star_symb, SFN_nbr from table 6.3.3.2-3 (TDD and FR1 scenario)
// - s_id is starting symbol of the PRACH occasion [0...14]
// - t_id is the first slot of the PRACH occasion in a system frame [0...80]
// - f_id: index of the PRACH occasion in the frequency domain
// - ul_carrier_id: UL carrier used for RA preamble transmission, hardcoded for NUL carrier
rnti_t nr_get_ra_rnti(uint8_t s_id, uint8_t t_id, uint8_t f_id, uint8_t ul_carrier_id)
{
  // 3GPP TS 38.321 Section 5.1.3
  rnti_t ra_rnti = 1 + s_id + 14 * t_id + 1120 * f_id + 8960 * ul_carrier_id;
  LOG_D(MAC, "f_id %d t_id %d s_id %d ul_carrier_id %d Computed RA_RNTI is 0x%04X\n", f_id, t_id, s_id, ul_carrier_id, ra_rnti);

  return ra_rnti;
}
```

<aside>

`TC-RNTI` allocated during RAR
`RA-RNTI` allocated from received PRACH

</aside>

## MSG2

![image.png](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/image%204.png)

![image.png](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/image%205.png)

`ra->ra_state` is `nrRA_Msg2` 

```c
      switch (ra->ra_state) {
        case nrRA_Msg2:
          nr_generate_Msg2(module_idP, CC_id, frameP, slotP, ra, DL_req, TX_req);
          break;
```

> 
> 
> 
> `nr_generate_Msg2` in a 5G gNB MAC (Medium Access Control) layer implementation, which handles the generation of the RA (Random Access) Msg2 response during the random access procedure. Key highlights of this function include:
> 
> 1. **DL Transmission Check**: Ensures that the downlink (DL) resources are available for transmission in the given slot.
> 2. **Response Window Validation**: Confirms that Msg2 is sent within the configured RA response window.
> 3. **Beam Allocation**: Allocates beamforming resources and validates the feasibility of the beam.
> 4. **TDA Feasibility**: Checks if the time domain allocation for Msg3 transmission is feasible and valid.
> 5. **Resource Allocation**: Identifies available VRBs (Virtual Resource Blocks) within the bandwidth part (BWP).
> 6. **PDCCH and PDSCH Setup**: Configures the PDCCH (Physical Downlink Control Channel) and PDSCH (Physical Downlink Shared Channel) PDUs (Protocol Data Units) for the Msg2 transmission.
> 7. **DCI Configuration**: Prepares the Downlink Control Information (DCI) payload for scheduling the Msg2 transmission.
> 8. **TB Size and MCS Selection**: Computes Transport Block Size (TBS) and Modulation and Coding Scheme (MCS) for Msg2.
> 9. **Contention Resolution**: Initiates the RA contention resolution timer for potential Msg3 processing.

## MSG3

![image.png](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/image%206.png)

### MSG3 Allocate

In `nr_generate_Msg2` allocate MSG3 

![image.png](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/image%207.png)

> // MSG3 sdu has 6 or 8 bytes
> 
> - **TDA Check**: The function ensures that the `Msg3_tda_id` is valid and within bounds (between 0 and 15).
> - **Time Domain Resource Allocation (TDA)**: Uses the TDA ID to extract the starting symbol and symbol length for Msg3 from the TDA list.
> - **Buffer Index Calculation**: Determines which part of the `vrb_map_UL` (virtual resource block map for uplink) corresponds to the Msg3 frame and slot.
> - **Resource Search**: The function iterates over the resource block (RB) map to find `msg3_nb_rb` (set to 8 RBs) of contiguous free RBs within the uplink BWP (Bandwidth Part). It ensures the start of Msg3 doesn't overlap with already occupied symbols.
> - **Bounds Check**: Checks and potentially adjusts the BWP start to fit within the active BWP if there is a mismatch in configurations.
> - **Allocation Completion**: Records the starting RB, number of RBs, and BWP start for Msg3.

Which function generate `ra->Msg3_tda_id`

![image.png](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/image%208.png)

 Start RA contention resolution **timer** in Msg3 transmission slot (current slot + K2)

```c
(ra_ContentionResolutionTimer + 1) * 8) << scs) + 2 * K2 //scs=2 k2=7 ra_ContentionResolutionTimer=7(SIB1)
//标准名称：ra_ContentionResolutionTimer。
//所在协议：3GPP TS 36.311，36.321。
//取值范围：sf8(8)、sf16(16)、sf24(24)、sf32(32)、sf40(40)、sf48(48)、sf56(56)、sf64(64)
```

### MSG3 Receive

![image.png](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/image%209.png)

```c
 [0m[NR_MAC]   Adding new UE context with RNTI 0x7e5f
 [0m[NR_MAC]   [gNB 0][RAPROC] PUSCH with TC_RNTI 0x7e5f received correctly, adding UE MAC Context RNTI 0x7e5f
 [0m [32m[NR_MAC]   [RAPROC] RA-Msg3 received (sdu_lenP 7)
```

**Handle MSG3 In `_nr_rx_sdu`** 

1. 接收和處理上行鏈路數據：函數接收來自PHY層的上行鏈路SDU，並進行相應處理。
2. UE狀態檢查：檢查接收到的數據是否來自已知的UE（User Equipment）或是正在進行隨機接入程序的新UE。
3. 信號質量評估：根據接收信號強度（**RSSI**）和上行鏈路CQI（Channel Quality Indicator）來評估信號質量。
4. 功率控制：根據信號質量計算**傳輸功率控制**（TPC）命令。 `Transmit power control`
5. 時間提前（Timing Advance）更新：如果需要，更新UE的時間提前值。
6. HARQ（Hybrid Automatic Repeat reQuest）處理：處理與HARQ相關的邏輯。
7. 隨機接入處理：處理隨機接入程序中的Msg3接收。
8. MAC PDU處理：調用 `nr_process_mac_pdu` 函數來處理接收到的MAC PDU。
9. 錯誤處理：處理可能的接收失敗情況，如檢測到上行鏈路失敗。

## MSG4

![image.png](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/image%2010.png)

![image.png](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/image%2011.png)

`prepare_initial_ul_rrc_message` 

1. **nr_rlc_activate_srb0 (CCCH)**
2. **nr_rlc_add_srb (DCCH)**

`nr_generate_Msg4` 

> 
> 
> - **Frame and Slot Check**: The function first checks if the current slot is a downlink slot and if Msg4 should be generated, based on the slot being active for downlink transmission.
> - **UE Search**: The function looks up the UE information for the corresponding `rnti` (Radio Network Temporary Identifier). If no UE information is found, the function logs an error and returns.
> - **Data Check**: The function checks if there is data to transmit (either on SRB0 or SRB1). If no data is available, it returns early.
> - **Beam Allocation**: It performs a beam allocation procedure to determine the appropriate beam for transmission, and returns if it is not successful.
> - **CCE Index**: The function finds a free CCE (Control Channel Element) for PDCCH transmission. If a CCE cannot be found, it logs an error and returns.
> - **TDA (Time Domain Assignment)**: It calculates the time-domain assignment for Msg4 and validates it. If invalid, the function returns.
> - **DMRS Info**: The DMRS (Demodulation Reference Signal) parameters are determined for the transmission.
> - **Transport Block Size Calculation**: Depending on the HARQ process and retransmission, the size of the transport block is computed. The algorithm scales the number of PRBs (Physical Resource Blocks) to fit the required transport block size.
> - **HARQ Process**: If there is an existing HARQ process for retransmission, the transport block size and feedback are managed. If no HARQ process is found, a new one is initiated.
> - **MAC PDU Creation**: The MAC PDU (Protocol Data Unit) is built, which includes the necessary subheaders and any required padding.
> - **DL TX Request**: A Downlink transmission request is created with the appropriate PDU, and the transport block is sent to the physical layer for transmission.
> - **Resource Allocation**: The function updates the VRB map to mark the resources used by Msg4, and the feedback process is initiated if needed.
> - **RA Procedure State**: After Msg4 is successfully prepared, the RA procedure state is updated to indicate the next step (either waiting for Msg4 acknowledgment or completing the process).

---

![image.png](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/image%2012.png)

> If we run the attacker, it will re-transmit MSG1 continually and the gNB noise will be increase to too high
> 
> 
> ```c
> gNB->measurements.prach_I0 = ((gNB->measurements.prach_I0 * 900) >> 10) + ((max_preamble_energy[0] * 124) >> 10);
> ```
> 
> 數學方程式：
> 
> $$
> prach\_I0_{new} = \frac{prach\_I0_{old} \times 900}{2^{10}} + \frac{max\_preamble\_energy[0] \times 124}{2^{10}}
> $$
> 
> ![image.png](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/image%2013.png)
> 
- The weights 1024900​≈0.8789 and 1024124​≈0.1211 sum to 1, ensuring the update represents a valid weighted average.
- This equation applies **exponential smoothing** to track the interference value.

<aside>

While the `noise+threshold` is bigger than UE’s Energy the MSG1 signal will not be detect

</aside>

[ Help ZX trace parameter in code](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/Help%20ZX%20trace%20parameter%20in%20code%2016510098314380d58ef1f6327756dc28.md)

[OAI PHY RX thread](ZX%E2%80%99s%20Random%20access%20MSC%2013c10098314380b7a8bec1c48067b44e/OAI%20PHY%20RX%20thread%2015a100983143800f9764f81d3d8bf7ff.md)