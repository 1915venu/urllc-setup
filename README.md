# URLLC Configuration Report
## OAI vs srsRAN Low Latency Analysis

**Date:** January 30, 2026  
**Author:** Venu  
**Purpose:** Configuration analysis for achieving URLLC (Ultra-Reliable Low-Latency Communication) with OAI gNB and B210 USRP

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [TDD Configuration Comparison](#tdd-configuration-comparison)
3. [OAI URLLC Parameters](#oai-urllc-parameters)
4. [srsRAN Low Latency Parameters](#srsran-low-latency-parameters)
5. [Parameter-by-Parameter Comparison](#parameter-by-parameter-comparison)
6. [srsRAN-Only URLLC Features](#srsran-only-urllc-features)

---

## Executive Summary

This report compares URLLC configuration parameters between **OpenAirInterface (OAI)** and **srsRAN Low Latency fork** to identify optimal settings for achieving low-latency 5G NR operation with B210 USRP hardware.

### Key Findings
- srsRAN Low Latency fork provides **additional tunable parameters** not available in OAI
- Both platforms support **2ms TDD periodicity** for reduced latency
- srsRAN offers explicit **K1/K2 timing control** for HARQ optimization
- OAI requires **sl_ahead=4** for B210 timing stability

---

## TDD Configuration Comparison

### 2ms TDD Pattern (DDDSU equivalent)

| Parameter | OAI Value | srsRAN Value | Description |
|-----------|-----------|--------------|-------------|
| TDD Period | 4 (2ms) | 4 (2ms) | Switching periodicity |
| DL Slots | 2 | 2 | Downlink-only slots |
| DL Symbols | 6 | 8 | DL symbols in special slot |
| UL Slots | 1 | 1 | Uplink-only slots |
| UL Symbols | 4 | 0 | UL symbols in special slot |

### Visual TDD Pattern

```
Slot:    |   0   |   1   |   2   |   3   |
         |  DL   |  DL   |DL/GP/UL|  UL   |
Symbols: |0-13   |0-13   |DL|GP|UL|0-13   |
```

---

## OAI URLLC Parameters

### TDD Configuration (gNB Config)
```c
tdd_ul_dl_configurationCommon:
  referenceSubcarrierSpacing: 1  // 30 kHz SCS
  pattern1:
    dl_UL_TransmissionPeriodicity: 4        // 2ms period
    nrofDownlinkSlots: 2
    nrofDownlinkSymbols: 6
    nrofUplinkSlots: 1
    nrofUplinkSymbols: 4
```

### MAC Scheduler Optimizations
```c
# Force continuous UL scheduling
ulsch_max_frame_inactivity = 0;   // Never stop UL scheduling

# Minimum UL grant size
min_grant_prb = 20;               // Minimum PRBs for UL

# LDPC decoder optimization
max_ldpc_iterations = 4;          // Reduce decoding iterations
```

### Critical Timing Parameters
```c
# B210 USRP requirements
sl_ahead = 4;                     // 4 slots TX preparation time

# Thread configuration
L1s:
  num_cc: 1
  pusch_proc_threads: 8
  prach_dtx_threshold: 160
```

### RACH Configuration
```c
prach_ConfigurationIndex: 159     // Format B4, any slot
msg1_SubcarrierSpacing: 1         // 30 kHz SCS
ra_ResponseWindow: 7              // 20 slots window
```

---

## srsRAN Low Latency Parameters

### TDD Configuration
```yaml
tdd_ul_dl_cfg:
  dl_ul_tx_period: 4      # 2ms TDD period
  nof_dl_slots: 2
  nof_dl_symbols: 8
  nof_ul_slots: 1
  nof_ul_symbols: 0
```

### Scheduling Optimizations
```yaml
cell_cfg:
  slicing:
    - sst: 1
      sched_cfg:
        sr_free_access_enable: false
        min_ul_grant_size: 512      # Min UL grant in bytes

  mac_cell_group:
    bsr_cfg:
      periodic_bsr_timer: 1         # 1ms BSR reporting

  pusch:
    min_k2: 2                       # Min PUSCH scheduling delay

  pucch:
    sr_period_ms: 2                 # SR period matching TDD
    min_k1: 2                       # Min HARQ feedback delay
```

### PHY Expert Parameters (Low Latency Fork Exclusive)
```yaml
expert_phy:
  max_proc_delay: 1        # Max DL processing delay (M-slot)
  radio_heads_prep_time: 3 # Radio head prep time (H-slot)
```

### Radio Configuration
```yaml
ru_sdr:
  srate: 23.040       # 23.04 MSPS for 15 MHz BW
  otw_format: sc12    # 12-bit reduces USB3 bandwidth
```

---

## Parameter-by-Parameter Comparison

| Feature | OAI Parameter | srsRAN Parameter | Purpose |
|---------|---------------|------------------|---------|
| **TDD Period** | `dl_UL_TransmissionPeriodicity = 4` | `dl_ul_tx_period: 4` | 2ms switching |
| **DL Slots** | `nrofDownlinkSlots = 2` | `nof_dl_slots: 2` | DL-only slots |
| **DL Symbols** | `nrofDownlinkSymbols = 6` | `nof_dl_symbols: 8` | Special slot DL |
| **UL Slots** | `nrofUplinkSlots = 1` | `nof_ul_slots: 1` | UL-only slots |
| **UL Symbols** | `nrofUplinkSymbols = 4` | `nof_ul_symbols: 0` | Special slot UL |
| **Slot Ahead** | `sl_ahead = 4` | N/A (auto) | TX prep time |
| **UL Scheduling** | `ulsch_max_frame_inactivity = 0` | N/A (always) | Continuous UL |
| **Min UL Grant** | `min_grant_prb = 20` | `min_ul_grant_size: 512` | Min allocation |
| **LDPC Iterations** | `max_ldpc_iterations = 4` | N/A | Fast decoding |
| **BSR Timer** | N/A | `periodic_bsr_timer: 1` | Buffer reporting |
| **SR Period** | N/A | `sr_period_ms: 2` | SR frequency |
| **K1 (HARQ)** | N/A | `min_k1: 2` | HARQ feedback |
| **K2 (PUSCH)** | N/A | `min_k2: 2` | UL scheduling |
| **Processing Delay** | N/A | `max_proc_delay: 1` | DL processing |
| **Radio Prep** | N/A | `radio_heads_prep_time: 3` | Radio timing |

---

## srsRAN-Only URLLC Features

These parameters are **only available in srsRAN Low Latency fork**:

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `sr_free_access_enable` | true/false | SR-free slice access |
| `min_ul_grant_size` | 512 bytes | Guaranteed min UL allocation |
| `max_proc_delay` | 1 slot | M-slot offset control |
| `radio_heads_prep_time` | 3 slots | H-slot offset control |
| `min_k1` | 2 | Minimum HARQ-ACK timing |
| `min_k2` | 2 | Minimum PUSCH scheduling |


---
