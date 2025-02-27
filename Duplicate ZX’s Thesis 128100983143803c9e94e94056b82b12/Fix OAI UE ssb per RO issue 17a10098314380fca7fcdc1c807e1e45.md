# Fix OAI UE ssb per RO issue

# My solution

在 5G 中，**association pattern**（關聯模式）指的是如何將 **SSB (Synchronization Signal Block)** 映射到 **RO (Random Access Occasion)** 的一個映射模式或規則。這種映射關係是為了實現設備與網絡之間的隨機接入（Random Access），並且需要根據標準配置或網絡的特定需求來設置。

```c
  const int required_nb_of_prach_occasion =
      multiple_ssb_per_ro ? ((ssb_list->nb_tx_ssb - 1) + ssb_rach_ratio) / ssb_rach_ratio : ssb_list->nb_tx_ssb * ssb_rach_ratio;
```

### 情況 1：多個 SSB 對應一個 RO (`multiple_ssb_per_ro == true`)

```c
required_nb_of_prach_occasion = ((ssb_list->nb_tx_ssb - 1) + ssb_rach_ratio) / ssb_rach_ratio;
```

### 情況 2：多個 RO 對應一個 SSB (`multiple_ssb_per_ro == false`)

```c
required_nb_of_prach_occasion = ssb_list->nb_tx_ssb * ssb_rach_ratio;
```

In this condition OAI used `prach_association_period->nb_of_frame;(Total number of frames included in the association pattern period (after mapping the SSBs and determining the real association pattern length)` this parameter is get from `map_ssb_to_ro()` instead of prach configuration table

```diff
commit 59b419d31c0ed8d6bea975c087b06d77f7ce25d7 (HEAD -> develop, origin/develop, origin/HEAD)
Author: Richard-yq <a2311496a@gmail.com>
Date:   Fri Jan 3 00:35:41 2025 +0800

    Replaced `prach_assoc_pattern->nb_of_frame` with `prach_assoc_pattern->prach_conf_period_list[0].nb_of_frame`. This aligns with the intended behavior where UE follows the PRACH configuration and PRACH period.
    
    Reasoning: The frame count should reflect the specific configuration period being referenced, ensuring accurate PRACH timing and association logic.

diff --git a/openair2/LAYER2/NR_MAC_UE/nr_ue_scheduler.c b/openair2/LAYER2/NR_MAC_UE/nr_ue_scheduler.c
index 557a393ce2..0c344c95ef 100644
--- a/openair2/LAYER2/NR_MAC_UE/nr_ue_scheduler.c
+++ b/openair2/LAYER2/NR_MAC_UE/nr_ue_scheduler.c
@@ -2123,7 +2123,7 @@ static int get_nr_prach_info_from_ssb_index(prach_association_pattern_t *prach_a
           ssb_info_p->mapped_ro[n_mapped_ro]->slot,
           prach_assoc_pattern->nb_of_frame);
     if ((slot == ssb_info_p->mapped_ro[n_mapped_ro]->slot) &&
-        (ssb_info_p->mapped_ro[n_mapped_ro]->frame == (frame % prach_assoc_pattern->nb_of_frame))) {
+        (ssb_info_p->mapped_ro[n_mapped_ro]->frame == (frame % prach_assoc_pattern->prach_conf_period_list[0].nb_of_frame))) {
       uint8_t prach_config_period_nb = ssb_info_p->mapped_ro[n_mapped_ro]->frame / prach_assoc_pattern->prach_conf_period_list[0].nb_of_frame;
       uint8_t frame_nb_in_prach_config_period = ssb_info_p->mapped_ro[n_mapped_ro]->frame % prach_assoc_pattern->prach_conf_period_list[0].nb_of_frame;
       prach_occasion_slot_p = &prach_assoc_pattern->prach_conf_period_list[prach_config_period_nb].prach_occasion_slot_map[frame_nb_in_prach_config_period][slot];
~
```

`map_ssb_to_ro()` multiplies`prach_association_period->nb_of_prach_conf_period`by`prach_conf_period->nb_of_frame`(from prach configuration table `x` )

```c
  prach_assoc_pattern->nb_of_assoc_period = 1; // WIP: only one possible association period
  prach_association_period->nb_of_frame = prach_association_period->nb_of_prach_conf_period * prach_conf_period->nb_of_frame;
  prach_assoc_pattern->nb_of_frame = prach_association_period->nb_of_frame;
```

# OAI Fixed

<aside>
💡

**Rework of NR UE RA procedures**

[https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/3150#e377ef41957175f4ac64a68e85f86096fa5bbabd](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/3150#e377ef41957175f4ac64a68e85f86096fa5bbabd)

[](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/3150/diffs?commit_id=013c3aaca3b2673a6d5a208fda36e0a7fd6caf85)

</aside>

Clone OAI repository and git checkout to `NR_UE_rework_RA` branch. (this change still not merge to `develop` branch)

![image.png](Fix%20OAI%20UE%20ssb%20per%20RO%20issue%2017a10098314380fca7fcdc1c807e1e45/image.png)

In **`nr_ue_prach_scheduler` the rework version invoke `is_prach_frame`**

```c
static bool is_prach_frame(frame_t frame, prach_occasion_info_t *prach_occasion_info, int association_periods)
{
  int config_period = prach_occasion_info->frame_info[0];
  int frame_period = config_period * association_periods;
  int frame_in_period = frame % frame_period;
  if (association_periods > 1) {
    int current_assoc_period = frame_in_period / config_period;
    if (current_assoc_period != prach_occasion_info->association_period_idx)
      return false;
    frame_in_period %= config_period;
  }
  if (frame_in_period == prach_occasion_info->frame_info[1]) // SFN % x == y (see Tables in Table 6.3.3.2 of 38.211)
    return true;
  else
    return false;
}
```

1. Check the `frame_in_period` and `config_period` and `association_periods` value (To be check)
    1. `frame_in_period` is 1prach_association_period
    2. `frame_period` is 2 
    3. `config_period` `prach_occasion_info->frame_info[0]` is 2 
    4. `association_periods` is 1
    5. `prach_occasion_info->frame_info[1]`  is 1
    
    ```
    [0m[NR_MAC]   PRACH scheduler: Frame 487, Period 2, Frame in period 1 config_period 2
    [0m[NR_MAC]   PRACH scheduler: Frame 487, Period 2, Frame in period 1 config_period 2
    [0m[NR_MAC]   PRACH scheduler: Frame 487, Period 2, Frame in period 1 config_period 2
    [0m[NR_MAC]   PRACH scheduler: Frame 487, Period 2, Frame in period 1 config_period 2
    [0m[NR_MAC]   PRACH scheduler:  frame_in_period 1, prach_occasion_info->frame_info[1] 1
    ```
    

---

They develop new function **`configure_prach_occasions` and `select_prach_occasion` to get the `x`** 

```c
  static void configure_prach_occasions(NR_UE_MAC_INST_t *mac, int scs){

			...
			...
				  int config_period = prach_info.x; // configuration period
				  ra->association_periods = 1;
				  int nb_eq_ssb = mac->ssb_list.nb_tx_ssb;
				  if (ra->ssb_ro_config.ssb_per_ro < 1)
				    nb_eq_ssb *= (int) (1 / ra->ssb_ro_config.ssb_per_ro);
				  int nb_eq_ro = num_ra_occasions_period;
			    LOG_I(NR_MAC, "Number of SSB %d Number of RO %d\n", nb_eq_ssb, nb_eq_ro);
				  if (ra->ssb_ro_config.ssb_per_ro > 1)
				    nb_eq_ro *= (int) ra->ssb_ro_config.ssb_per_ro;
				  while (nb_eq_ssb > nb_eq_ro) {
				    // not enough PRACH occasions -> need to increase association period
				    ra->association_periods <<= 1;
				    AssertFatal(ra->association_periods * config_period <= 16,
				                "Cannot find an association period for %d SSB and %d RO with %f SSB per RO\n",
				                mac->ssb_list.nb_tx_ssb,
				                num_ra_occasions_period,
				                ra->ssb_ro_config.ssb_per_ro);
				    nb_eq_ro <<= 1; // doubling the association period -> doubling ROs
				  }
				  LOG_I(NR_MAC, "PRACH configuration period %d association period %d\n", config_period, ra->association_periods);
				
```

one → Number of SSB 1 Number of RO 3

onehalf → Number of SSB 2 Number of RO 3

```c
    if ((curr_mask >> (31 - (ssb_index % 32))) & 0x01) {
      ssb_list->nb_ssb_per_index[ssb_index] = ssb_list->nb_tx_ssb;
      ssb_list->nb_tx_ssb++;
    }
```

```
[0m[NR_MAC]   RA occasion 0: slot 19 start symbol 0 fd occasion 0
[0m[NR_MAC]   RA occasion 1: slot 19 start symbol 4 fd occasion 0
[0m[NR_MAC]   RA occasion 2: slot 19 start symbol 8 fd occasion 0
```

<aside>
💡

OAI re-write **`configure_prach_occasions`** and  **`select_prach_occasion` to solve ssb_per_ro issue** https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/3150/diffs?commit_id=678876bccfb9111e810f4a4bf6f5e24c060df061

```c
// 38.321 Section 5.1.2 Random Access Resource selection
void ra_resource_selection(NR_UE_MAC_INST_t *mac)
{
  configure_ra_preamble(mac);
  const NR_UE_UL_BWP_t *current_UL_BWP = mac->current_UL_BWP;
  int scs = nr_get_prach_mu(current_UL_BWP->msgA_ConfigCommon_r16, current_UL_BWP->rach_ConfigCommon);
  configure_prach_occasions(mac, scs);
  ra_preamble_msga_transmission(&mac->ra, scs);
}
```

**`get_ssb_ro_preambles_4step`移除了**`multiple_ssb_per_ro`**的布林值判斷，改用小數或正整數表示 SSB 與 RO 的對應關係**

**harmonization of functions to get info from PRACH config tables**

</aside>

---

## Rebase branch to `NR_UE_rework_RA` to verify that ZX's attacker functions correctly

<aside>
👀

See deatil:

[](https://github.com/Richard-yq/OAI-UE-MSG1-attacker/commit/ea3097af7a83bcafb83280ccac90a95c6dc9c5b2#diff-fe4b1ca4d3ee1cc83e809131aa1b392d0ef7eed085a14f310bb7500a07889b18R2335)

</aside>

```bash
  554  cd richard/OAI-UE-MSG1-attacker/
  555  git branch rework_UE
  556  git checkout -b rework_UE 
  557  git push origin rework_UE 
  558  git branch -r
  559  git remote -v
  560  git remote add oai https://gitlab.eurecom.fr/oai/openairinterface5g.git
  561  git remote -v
  562  git fetch oai
  563  git status 
  564  git log
  565  git checkout 10218d3e7cf0d6f05c8c9aa8616974edfdd8d50c
  566  git rebase oai/NR_UE_rework_RA
  567  git status 
  568  git log
  569  git checkout -b rework_UE_2
  570  git push origin rework_UE_2 
```

確認目前的SSB配置是否正確了

### 問題1

After SIB1 decoded, the OAI UE encounters a Floating point exception

![image.png](Fix%20OAI%20UE%20ssb%20per%20RO%20issue%2017a10098314380fca7fcdc1c807e1e45/image%201.png)

Use gdb to trace

```bash
sudo gdb --args ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 -E --ue-fo-compensation
```

```bash
[PHY]   Resynchronizing RX by 203936 samples
[HW]   received write reorder clear context
[NR_RRC]   SIB1 decoded

Thread 4 "Tpool2_-1" received signal SIGFPE, Arithmetic exception.
[Switching to Thread 0x7ffff5c00640 (LWP 178816)]
0x000055555596a3c2 in is_prach_frame (association_periods=0, prach_occasion_info=<optimized out>, frame=732) at /home/oaiue/richard/OAI-UE-MSG1-attacker/openair2/LAYER2/NR_MAC_UE/nr_ue_scheduler.c:2339
2339      int frame_in_period = frame % frame_period;
```

```bash
[HW]   received write reorder clear context
[NR_RRC]   SIB1 decoded

Thread 4 "Tpool2_-1" received signal SIGFPE, Arithmetic exception.
[Switching to Thread 0x7ffff5c00640 (LWP 178816)]
0x000055555596a3c2 in is_prach_frame (association_periods=0, prach_occasion_info=<optimized out>, frame=732) at /home/oaiue/richard/OAI-UE-MSG1-attacker/openair2/LAYER2/NR_MAC_UE/nr_ue_scheduler.c:2339
2339      int frame_in_period = frame % frame_period;
(gdb) backtrace
#0  0x000055555596a3c2 in is_prach_frame (association_periods=0, prach_occasion_info=<optimized out>, frame=732)
    at /home/oaiue/richard/OAI-UE-MSG1-attacker/openair2/LAYER2/NR_MAC_UE/nr_ue_scheduler.c:2339
#1  nr_ue_prach_scheduler (mac=0x7ffff791c010, frameP=732, slotP=7) at /home/oaiue/richard/OAI-UE-MSG1-attacker/openair2/LAYER2/NR_MAC_UE/nr_ue_scheduler.c:2363
#2  0x000055555596f997 in nr_ue_ul_scheduler (mac=mac@entry=0x7ffff791c010, ul_info=ul_info@entry=0x7ffff5bff3d0)
    at /home/oaiue/richard/OAI-UE-MSG1-attacker/openair2/LAYER2/NR_MAC_UE/nr_ue_scheduler.c:1612
#3  0x000055555590d8e3 in nr_ue_ul_indication (ul_info=0x7ffff5bff3d0) at /home/oaiue/richard/OAI-UE-MSG1-attacker/openair2/NR_UE_PHY_INTERFACE/NR_IF_Module.c:1208
#4  0x00005555558356bf in processSlotTX (arg=0x7fffc015e9d0) at /home/oaiue/richard/OAI-UE-MSG1-attacker/executables/nr-ue.c:559
#5  0x0000555555b57679 in worker_thread (arg=<optimized out>) at /home/oaiue/richard/OAI-UE-MSG1-attacker/common/utils/threadPool/thread-pool.c:93
#6  0x00007ffff7294ac3 in start_thread (arg=<optimized out>) at ./nptl/pthread_create.c:442
#7  0x00007ffff7326850 in clone3 () at ../sysdeps/unix/sysv/linux/x86_64/clone3.S:81
```

1. The `frame_in_period` variable, being an integer, may cause an arithmetic overflow
2. **添加防範檢查** 在 `frame % frame_period` 運算之前檢查 `frame_period` 是否為 0：
    
    ![image.png](Fix%20OAI%20UE%20ssb%20per%20RO%20issue%2017a10098314380fca7fcdc1c807e1e45/image%202.png)
    

```bash
Error: frame_period is 0
DEBUG: association_periods=0, frame=153
Error: frame_period is 0
DEBUG: association_periods=0, frame=153
Error: frame_period is 0
DEBUG: association_periods=0, frame=153
Error: frame_period is 0
^CDEBUG: association_periods=0, frame=154
Error: frame_period is 0
DEBUG: association_periods=0, frame=154
Error: frame_period is 0

DEBUG: association_periods=0, frame=83 , config_period=0
Error: frame_period is 0
```

解決方法：

return directly

### 問題2

這個判斷式進不去

```bash
  if (ue->prach_vars[gNB_id]->active) {
```

```c
static void nr_ue_prach_scheduler(NR_UE_MAC_INST_t *mac, frame_t frameP, int slotP)
{
```

![image.png](Fix%20OAI%20UE%20ssb%20per%20RO%20issue%2017a10098314380fca7fcdc1c807e1e45/image%203.png)

ZX’s attacker only pass `if (is_nr_UL_slot(tdd_config, slotP, mac->frame_type))`  this condition

```bash
Assertion (1==0) failed!
In generate_nr_prach() /home/oaiue/richard/OAI-UE-MSG1-attacker/openair1/PHY/NR_UE_TRANSPORT/nr_prach.c:308
Unknown PRACH format ID 0
```

解決方法 ：

Hardcode   `prach_fmt_id`  = 8;

<aside>
👀

See deatil:

[](https://github.com/Richard-yq/OAI-UE-MSG1-attacker/commit/ea3097af7a83bcafb83280ccac90a95c6dc9c5b2#diff-fe4b1ca4d3ee1cc83e809131aa1b392d0ef7eed085a14f310bb7500a07889b18R2335)

</aside>

---

## Outcome

1.         ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR                = 4

```bash
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 3, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 3.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 4, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 4.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 5, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 5.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 6, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 6.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 7, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 7.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 8, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 8.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 9, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 9.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 10, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 10.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 11, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 11.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 12, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 12.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
```

1.  ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR                = 3

```bash
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 154, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 154.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 155, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 155.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 156, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 156.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 157, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 157.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 158, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 158.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 159, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 159.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 160, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 160.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 161, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 161.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
^C[NR_MAC]   PRACH scheduler: Selected RO Frame 162, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 162.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
```

1.  ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR                = 2

```bash
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 941, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 941.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 942, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 942.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 943, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 943.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 944, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 944.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 945, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 945.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 946, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 946.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
^C
```

1. ssb_perRACH_OccasionAndCB_PreamblesPerSSB_PR                = 1

```bash
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 291, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 291.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 292, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 292.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 293, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 293.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 294, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 294.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 295, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 295.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 296, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 296.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 297, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 297.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 298, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 298.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 299, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 299.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 300, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 300.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 301, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 301.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 302, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 302.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 303, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 303.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 304, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 304.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 305, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 305.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
[NR_MAC]   PRACH scheduler: Selected RO Frame 306, Slot 19, Symbol 0, Fdm 0
[PHY]   PRACH [UE 0] in frame.slot 306.19, placing PRACH in position 1804, Msg1/MsgA-Preamble frequency start 0 (k1 0), preamble_offset 0, first_nonzero_root_idx 0
PRACH format ID 0
```