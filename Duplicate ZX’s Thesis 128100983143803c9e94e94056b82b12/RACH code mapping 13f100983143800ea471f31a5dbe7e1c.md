# RACH code mapping

<aside>

[](https://gitlab.eurecom.fr/oai/openairinterface5g.git)

[](https://hackmd.io/-QkBAyzvRiGjXfzmM0a-MA?view#ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR)

[https://hackmd.io/-QkBAyzvRiGjXfzmM0a-MA?view#實驗](https://hackmd.io/-QkBAyzvRiGjXfzmM0a-MA?view#%E5%AF%A6%E9%A9%97)

[https://www.linkedin.com/pulse/nr-rach-process-syed-mohiuddin/](https://www.linkedin.com/pulse/nr-rach-process-syed-mohiuddin/)

[5G ---SSB与preamble occasion - shiyuan310 - 博客园](https://www.cnblogs.com/beilou310/p/11309186.html)

</aside>

## Background

![image.png](RACH%20code%20mapping%2013f100983143800ea471f31a5dbe7e1c/image.png)

<aside>

`ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR`  

指的是一個SSB對應幾個RO，幫我找這個參數在OAI **frequecy**/time resource怎麼mapping的

`ssPBCH_BlockPower`

指的是SSB在gNB傳輸的power大小，幫我確認這個參數在OAI code用到的相關函數 

實際上有沒有用到？

---

ZX:

If we modify `ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR` to 3 (half) and increase `ssPBCH_BlockPower`, the attacker might fail.

Plz check OAI how to implement

</aside>

`ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR` have 8 cases 

1=oneeighth,2=onefourth,3=half,4=one,5=two,6=four,7=eight,8=sixteen

---

// This routine implements RA preamble configuration according to
// section 5.1 (Random Access procedure) of 3GPP TS 38.321 version 16.2.1 Release 16

```c
void ssb_rach_config(RA_config_t *ra, NR_PRACH_RESOURCES_t *prach_resources, NR_RACH_ConfigCommon_t *nr_rach_ConfigCommon)
{
  // Determine the SSB to RACH mapping ratio
  // =======================================

  NR_RACH_ConfigCommon__ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR ssb_perRACH_config = nr_rach_ConfigCommon->ssb_perRACH_OccasionAndCB_PreamblesPerSSB->present;
  bool multiple_ssb_per_ro; // true if more than one or exactly one SSB per RACH occasion, false if more than one RO per SSB
  uint8_t ssb_rach_ratio; // Nb of SSBs per RACH or RACHs per SSB
  int total_preambles_per_ssb;
  uint8_t ssb_nb_in_ro;
```

<aside>

Two case 

1. (`TRUE`)more than one or exactly one SSB per RACH occasion
2. (`FALSE`)more than one RO per SSB

---

`ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR` = 3 

  multiple_ssb_per_ro = **false**; → **more than one RO per SSB** 

<aside>

**2 RO in one SSB**

</aside>

`ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR` = 4

  multiple_ssb_per_ro = **false**; → **exactly one SSB per RACH occasion**

<aside>

**1 RO in one SSB**

</aside>

</aside>

UE handle ssb and map to Random access occasion

```c
// Map the transmitted SSBs to the ROs and create the association pattern according to the config
static void map_ssb_to_ro(NR_UE_MAC_INST_t *mac)
{
  // Map SSBs to PRACH occasions
  // WIP: Assumption: No PRACH occasion is rejected because of a conflict with SSBs or TDD_UL_DL_ConfigurationCommon schedule
  NR_RACH_ConfigCommon_t *setup = mac->current_UL_BWP->rach_ConfigCommon;
  NR_RACH_ConfigCommon__ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR ssb_perRACH_config = setup->ssb_perRACH_OccasionAndCB_PreamblesPerSSB->present;
```

**find_SSB_and_RO_available**

**get_nr_prach_occasion_info_from_index**

```c
//TDD FR1
table_6_3_3_2_3_prachConfig_Index
// Table 6.3.3.2-3: Random access configurations for FR1 and unpaired spectrum
static const int64_t table_6_3_3_2_3_prachConfig_Index[256][9] = {
    // format,     format,      x,         y,     SFN_nbr,   star_symb,   slots_sfn,  occ_slot,  duration
    {0, -1, 16, 1, 512, 0, 1, 1, 0}, // (subrame number 9)
    {0, -1, 8, 1, 512, 0, 1, 1, 0}, // (subrame number 9)
    ...
```

TODO: Try to open this log

```c
LOG_D(NR_MAC, "Total available RO %d, num of active SSB %d: unused RO = %d ...", ...);
```

<aside>

Idea:

If we modify `ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR` (the "**present**" field of ssb_perRACH_OccasionAndCB_PreamblesPerSSB), should we also modify `ssb_perRACH_OccasionAndCB_PreamblesPerSSB` at the same time?

</aside>

<aside>

Using this code for example:

```c
  switch (ssb_perRACH_OccasionAndCB_PreamblesPerSSB->present){
    case NR_RACH_ConfigCommon__ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR_oneEighth:
      cc->cb_preambles_per_ssb = 4 * (ssb_perRACH_OccasionAndCB_PreamblesPerSSB->choice.oneEighth + 1);
      break;
    case NR_RACH_ConfigCommon__ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR_oneFourth:
      cc->cb_preambles_per_ssb = 4 * (ssb_perRACH_OccasionAndCB_PreamblesPerSSB->choice.oneFourth + 1);
      break;
    case NR_RACH_ConfigCommon__ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR_oneHalf:
      cc->cb_preambles_per_ssb = 4 * (ssb_perRACH_OccasionAndCB_PreamblesPerSSB->choice.oneHalf + 1);
      break;
    case NR_RACH_ConfigCommon__ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR_one:
      cc->cb_preambles_per_ssb = 4 * (ssb_perRACH_OccasionAndCB_PreamblesPerSSB->choice.one + 1);
      break;
```

If the present is “4” (**1 RO in one SSB**)

`cb_preambles_per_ssb` = 4 * (`ssb_perRACH_OccasionAndCB_PreamblesPerSSB`(15) +1)

`CB-PreamblesPerSSB` indicate that how many Contention-Based Preambles are mapped on each SSB 0 ~ 64

</aside>

![image.png](RACH%20code%20mapping%2013f100983143800ea471f31a5dbe7e1c/image%201.png)

> prach_msg1_FDM
> 

![image.png](RACH%20code%20mapping%2013f100983143800ea471f31a5dbe7e1c/image%202.png)

---

## Implement in Code

Next step: Check OAI gNB allocate PRACH resource code

```c
void schedule_nr_prach(module_id_t module_idP, frame_t frameP, sub_frame_t slotP)
```

<aside>

1. Ensure slot is UL slot

```c
  if (is_nr_UL_slot(scc->tdd_UL_DL_ConfigurationCommon, slotP, cc->frame_type)) {

```

1. Get the prach information from `prach_index`

`format`, `SFN_nbr`,   `star_symb`,   `slots_sfn`,  `occ_slot`,

```c
    // prach is scheduled according to configuration index and tables 6.3.3.2.2 to 6.3.3.2.4
    if (get_nr_prach_info_from_index(config_index,                                     (int)frameP,
                                     (int)slotP,
                                     scc->downlinkConfigCommon->frequencyInfoDL->absoluteFrequencyPointA,
                                     mu,
                                     cc->frame_type,
                                     &format,
                                     &start_symbol,
                                     &N_t_slot,
                                     &N_dur,
                                     &RA_sfn_index,
                                     &N_RA_slot,
                                     &config_period))

```

1. **Calculate prach occasion ID**

```c
      prach_occasion_id = (((frameP % (cc->max_association_period * config_period))/config_period) * cc->total_prach_occasions_per_config_period) +
                          (RA_sfn_index + slot_index) * N_t_slot * fdm + td_index * fdm + fdm_index;

```

1. Get SSB per RO from table 

```c
  float num_ssb_per_RO = ssb_per_rach_occasion[cfg->prach_config.ssb_per_rach.value];

	static const float ssb_per_rach_occasion[8] = {0.125, 0.25, 0.5, 1, 2, 4, 8};

```

1. **Allocate Beam**

```c
  if(num_ssb_per_RO <= 1) {
    ...
    beam = beam_allocation_procedure(&gNB->beam_info, frameP, slotP, beam_index, nr_slots_per_frame[mu]);
    AssertFatal(beam.idx >= 0, "Cannot allocate PRACH corresponding to %d SSB transmitted in any available beam\\n", n_ssb + 1);
  }

```

1. Configuration PRACH PDU struct (FAPI)

```c
  if(td_index == 0) {
    UL_tti_req->pdus_list[UL_tti_req->n_pdus].pdu_type = NFAPI_NR_UL_CONFIG_PRACH_PDU_TYPE;
    nfapi_nr_prach_pdu_t  *prach_pdu = &UL_tti_req->pdus_list[UL_tti_req->n_pdus].prach_pdu;
    memset(prach_pdu,0,sizeof(nfapi_nr_prach_pdu_t));
    UL_tti_req->n_pdus+=1;

    prach_pdu->phys_cell_id = *scc->physCellId;
    prach_pdu->num_prach_ocas = N_t_slot;
    ...
  }

```

</aside>

<aside>

Print log

```c
            LOG_D(NR_MAC,
                  "Frame %d, Slot %d: Prach Occasion id = %u  fdm index = %u start symbol = %u slot index = %u subframe index = %u \n",
                  frameP,
                  slotP,
                  prach_occasion_id,
                  prach_pdu->num_ra,
                  prach_pdu->prach_start_symbol,
                  slot_index,
                  RA_sfn_index);
```

```c

  LOG_I(NR_MAC, "[richard]Frame %d, Slot %d: Prach Occasion id = %d ssb per RO = %f number of active SSB %u index = %d fdm %u symbol index %u freq_index %u total_RApreambles %u\n",
        frameP, slotP, prach_occasion_id, num_ssb_per_RO, num_active_ssb, index, fdm, start_symbol_index, freq_index, total_RApreambles);

```

LOG (half)

```bash
[0m[NR_MAC]   [richard1] Frame 308, Slot 19: Prach Occasion id = 0 ssb per RO = 0.500000 number of active SSB 1 index = 0 fdm 1 symbol index 0 freq_index 0 total_RApreambles 64
[0m[32m[NR_MAC]   UE 138d: 309.7 Generating RA-Msg2 DCI, RA RNTI 0x10b, state 1, CoreSetType 0, RAPID 30
[0m[NR_MAC]   UE 138d: Msg3 scheduled at 309.17 (309.7 k2 7 TDA 3)
[0m[NR_MAC]   Starting RA Contention Resolution timer with 64 ms + 2 * 7 K2 (142 slots) duration
[0m[NR_MAC]    30910: RA RNTI c16a CC_id 0 Scheduling retransmission of Msg3 in (309,17)
[0m[NR_MAC]   Starting RA Contention Resolution timer with 64 ms + 2 * 7 K2 (142 slots) duration
[0m[PHY]   Initializing nFAPI for ULSCH, harq_pid 0, layers 1
[0m[PHY]   Initializing nFAPI for ULSCH, harq_pid 0, layers 1
[0m[NR_MAC]   [richard2] Frame 310, Slot 19: Prach Occasion id = 0  fdm index = 0 start symbol = 0 slot index = 0 subframe index = 0 
```

LOG (original)

```bash
[0m[NR_MAC]   [richard1] Frame 71, Slot 19: Prach Occasion id = 0 ssb per RO = 1.000000 number of active SSB 1 index = 0 fdm 1 symbol index 0 freq_index 0 total_RApreambles 64
[0m[32m[NR_MAC]   UE 35d0: 72.7 Generating RA-Msg2 DCI, RA RNTI 0x10b, state 1, CoreSetType 0, RAPID 14
[0m[NR_MAC]   UE 35d0: Msg3 scheduled at 72.17 (72.7 k2 7 TDA 3)
[0m[NR_MAC]   Starting RA Contention Resolution timer with 64 ms + 2 * 7 K2 (142 slots) duration
[0m[PHY]   Initializing nFAPI for ULSCH, harq_pid 0, layers 1
[0m[NR_MAC]   [richard2] Frame 73, Slot 19: Prach Occasion id = 0  fdm index = 0 start symbol = 0 slot index = 0 subframe index = 0 
```

</aside>

## Conclusion

[https://viewer.diagrams.net/?border=0&tags=%7B%7D&lightbox=10&highlight=0000ff&edit=_blank&layers=1&nav=10&title=LAB.drawio#Uhttps%3A%2F%2Fdrive.google.com%2Fuc%3Fid%3D1epnjxR2YKsmGwDnT3gXcDMIl2_D8vQgP%26export%3Ddownload#%7B%22pageId%22%3A%22evL6ZDn-zJhji04FCCIn%22%7D](https://viewer.diagrams.net/?border=0&tags=%7B%7D&lightbox=10&highlight=0000ff&edit=_blank&layers=1&nav=10&title=LAB.drawio#Uhttps%3A%2F%2Fdrive.google.com%2Fuc%3Fid%3D1epnjxR2YKsmGwDnT3gXcDMIl2_D8vQgP%26export%3Ddownload#%7B%22pageId%22%3A%22evL6ZDn-zJhji04FCCIn%22%7D)

<aside>

### Why the attacker success possibility is almost 100%

1. Depend on antenna power
    1. Attacker is closer the gNB
    2. gNB behavior
        1. After execute attacker, gNB enter `handle_nr_rach` every frame
            
            ![image.png](RACH%20code%20mapping%2013f100983143800ea471f31a5dbe7e1c/image%203.png)
            
        2. If have UE send preamble the `max_preamble_energy[0]` will be increase to five hundreds (**obtain from** **`rx_nr_prach`)**
            
            ![image.png](RACH%20code%20mapping%2013f100983143800ea471f31a5dbe7e1c/image%204.png)
            
        3. For the **`rx_nr_prach` see** 
            
            [Review RACH thesis ](Review%20RACH%20thesis%20131100983143805cbd46f3b0255919e9.md)
            
            ![image.png](RACH%20code%20mapping%2013f100983143800ea471f31a5dbe7e1c/image%205.png)
            
</aside>