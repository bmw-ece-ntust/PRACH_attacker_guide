# Help ZX trace parameter in code

### `min_rxtxtime`

```c
#define GNB_CONFIG_STRING_MINRXTXTIME                   "min_rxtxtime"

...

#define NR_UE_CAPABILITY_SLOT_RX_TO_TX (3)
#define DURATION_RX_TO_TX (NR_UE_CAPABILITY_SLOT_RX_TO_TX + NTN_UE_Koffset)

```

```c
    AssertFatal((k2 + delta) > DURATION_RX_TO_TX,
                "Slot offset (%ld) for Msg3 needs to be higher than DURATION_RX_TO_TX (%d). Please set min_rxtxtime at least to %d in gNB config file or gNBs.[0].min_rxtxtime=%d via command line.\n",
                k2,
                DURATION_RX_TO_TX,
                DURATION_RX_TO_TX,
                DURATION_RX_TO_TX);
```

```c
    AssertFatal(k2 > DURATION_RX_TO_TX,
                "Slot offset K2 (%ld) needs to be higher than DURATION_RX_TO_TX (%d). Please set min_rxtxtime at least to %d in gNB config file or gNBs.[0].min_rxtxtime=%d via command line.\n",
                k2,
                DURATION_RX_TO_TX,
                DURATION_RX_TO_TX,
                DURATION_RX_TO_TX);
```

### `min_rxtxtime` is relate to K2

![image.png](Help%20ZX%20trace%20parameter%20in%20code%2016510098314380d58ef1f6327756dc28/image.png)

- Develop branch (Add NTN)
    
    ```c
      const int K2 = nr_mac->radio_config.minRXTXTIME + get_NTN_Koffset(scc);
    ```
    
    ```c
      config.minRXTXTIME = *GNBParamList.paramarray[0][GNB_MINRXTXTIME_IDX].iptr;
    ```
    
- ZX’s gNB branch
    
    ```c
      nr_rrc_config_ul_tda(scc, minRXTXTIME);
    ...
    void nr_rrc_config_ul_tda(NR_ServingCellConfigCommon_t *scc, int min_fb_delay){
      const int k2 = min_fb_delay;
    ```