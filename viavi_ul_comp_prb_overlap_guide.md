# VIAVI UE Emulator Issues & Architecture Guide: Uplink CoMP Reception in PRB Overlapping Regions

## 1. Overview & Common Failure Symptoms

When testing **Uplink Coordinated Multi-Point (UL CoMP) Reception** (or Multi-TRP UL reception) with a **VIAVI UE Emulator** (such as the TM500 or E500), issues in a **Physical Resource Block (PRB) overlapping region** typically stem from synchronization mismatches, power control conflicts, DMRS (Demodulation Reference Signal) collisions, or fronthaul/channel model configuration errors.

In a PRB-overlapping UL CoMP scenario—where two UEs or two spatial layers share the exact same time-frequency PRBs across cooperating cells/TRPs, or a single UE transmits to two TRPs using overlapping PRB allocations—the UE emulator and gNB/eNB typically exhibit:

* **High PUSCH BLER / CRC Failures:** The cooperating reception points fail to separate or coherently combine the overlapping uplink streams.
* **HARQ Retransmission Loops:** Excessive NACKs occur even when simulated path loss and SNR appear favorable.
* **UL Throughput Degradation:** The scheduler falls back to conservative Modulation and Coding Schemes (MCS) or drops out of CoMP Joint Reception (JR) mode into non-CoMP single-TRP reception.
* **DMRS Channel Estimation Errors:** Poor SINR reporting on the uplink DMRS, leading to inaccurate Interference Rejection Combining (IRC) or joint processing at the baseband unit (BBU/O-DU).

---

## 2. Primary Root Causes & Configuration Pitfalls

* **Timing Advance (TA) and Propagation Delay Mismatch**
  * **Cause:** In UL CoMP Joint Reception, signals from a UE arrive at two different TRPs with different propagation delays. If the VIAVI channel simulator profile applies asymmetrical delays between the UE and the cooperating cells that exceed the Cyclic Prefix (CP) window, Inter-Symbol Interference (ISI) occurs across the overlapping PRBs.
  * **Impact:** Coherent combining breaks down completely when PRBs overlap between a cell-edge UE and a cell-center UE.
* **DMRS Port and Scrambling ID (`n_SCID`) Collisions**
  * **Cause:** When multiple UEs are scheduled on overlapping PRBs in UL MU-MIMO / CoMP, the network relies on orthogonal DMRS ports or distinct scrambling identities (`n_SCID` / `dmrs-UplinkForPUSCH-MappingTypeA/B`) to separate the channels.
  * **Impact:** If the VIAVI UE script uses hardcoded or default DMRS initialization parameters rather than dynamically following the RRC reconfiguration from the gNB/eNB, the receiver cannot decouple the overlapping PRBs.
* **Uplink Power Control ($P_0$ and $\alpha$) Imbalance**
  * **Cause:** In a PRB overlapping region, a UE transmitting at high power to reach a distant cooperating TRP can drown out a nearby UE sharing the same PRBs (the "near-far" problem) if fractional path-loss compensation ($\alpha$) is misconfigured.
  * **Impact:** If the VIAVI UE emulator does not accurately apply closed-loop TPC (Transmit Power Control) commands per TRP/SRS resource set, the dynamic range at the RU/DU receiver is exceeded.
* **Non-Contiguous PRB / Fronthaul (O-RAN 7.2x) Support**
  * **Cause:** In disaggregated O-RAN setups using the TM500/E500 over fronthaul (e.g., Option 7-2x), overlapping or fragmented PRB allocations require specific fronthaul license capabilities (such as VIAVI's non-contiguous PRB support modules) and strict eCPRI section extension handling.
  * **Impact:** PRBs in the overlapping boundary may be truncated or dropped on the fronthaul interface before reaching the O-DU equalizer.
* **SRS (Sounding Reference Signal) Configuration Mismatch**
  * **Cause:** Accurate UL CoMP scheduling depends on fresh Channel State Information (CSI) derived from UE SRS transmissions. If the UE emulator's SRS bandwidth, comb offset, or periodicity does not align with the cooperating cell's SRS measurement window, the scheduler assigns overlapping PRBs to UEs that are actually spatially correlated.

---

## 3. General Troubleshooting & Verification Checklist

* **Verify Channel Model Delays:** Check the VIAVI embedded fading/channel emulator settings between the UE and each TRP. Ensure the differential delay between Cell A and Cell B stays well within the normal Cyclic Prefix duration (e.g., < 4.7 μs for 15 kHz SCS, or proportionally shorter for 30 kHz SCS).
* **Inspect RRC & DCI Logs on the TM500/E500:**
  * Confirm that the UE is correctly parsing DCI Format 0_1 / 0_2 fields for **SRS Resource Indicator (SRI)** and **Antenna Ports / DMRS sequence initialization**.
  * Verify that the virtual UEs in the overlapping PRB zone are assigned orthogonal DMRS ports or distinct scrambling IDs.
* **Check Uplink Power Headroom (PHR) & TPC:** Review the TM500/E500 PHY layer logs to confirm that UE transmit power is tracking the gNB's TPC commands and not hitting max power ($P_{\text{CMAX}}$) prematurely.
* **Isolate CoMP Modes:** Test **Coordinated Scheduling/Beamforming (CS/CB)** first (where PRBs do not overlap) before enabling **Joint Reception (JR)** with PRB overlap. This isolates basic multi-cell RF/timing synchronization from baseband IRC/equalizer issues.
* **Check Software Release Notes:** Ensure your TM500/E500 software build supports the specific 3GPP Release feature set being tested (e.g., Release 16/17 Multi-TRP PUSCH repetition or simultaneous multi-panel UL transmission).

---

## 4. General UL CoMP PRB Overlapping Architecture & Resource Grid

In **Uplink CoMP Joint Reception (UL CoMP-JR)**, multiple cells (or TRPs) cooperate to receive uplink transmissions simultaneously. The **PRB overlapping region** occurs when UEs located in the cell-edge overlap zone transmit on the **exact same Physical Resource Blocks (PRBs)** at the same time, relying on spatial separation (MU-MIMO), orthogonal DMRS ports, and joint baseband processing to separate the signals.

### 4.1 Spatial & Network Topology (Cell-Edge Overlap)

In this example, **UE 1** (served primarily by Cell A) and **UE 2** (served primarily by Cell B) are both in the overlapping coverage region. Instead of avoiding interference by splitting frequencies, both UEs transmit on shared PRBs (PRBs 4–7), and both TRPs forward the raw baseband IQ data to a centralized **BBU / O-DU** for joint equalization.

```text
      [ Serving Cell A / TRP 1 ]                 [ Cooperating Cell B / TRP 2 ]
         (e.g., PCI = 101)                             (e.g., PCI = 102)

                 |                                             |
         Receives: h11*x1 + h12*x2                     Receives: h21*x1 + h22*x2

                 |                                             |
                 +---------------------\ /---------------------+
                                        |
                             ( Fronthaul / Backhaul )
                                        |
                                        v
                        +-------------------------------+
                        |   Centralized BBU / O-DU      |
                        |  Joint Reception Equalizer    |
                        |  (IRC / MMSE Multi-Cell Comb) |
                        +-------------------------------+
                                  /                                            v             v
                         Decoded UE 1       Decoded UE 2
                        (PRBs 0 to 7)      (PRBs 4 to 11)

================================================================================
                        RF COVERAGE & OVERLAP ZONE

      ( . . . . . . . Cell A Coverage . . . . ( Overlap Zone ) . . . . Cell B Coverage . . . . . )

                                              |              |
                                           [UE 1]         [UE 2]
                                       (PRBs 0 - 7)   (PRBs 4 - 11)

                                              |              |
                                              +------+-------+
                                                     |
                                         Overlapping PRBs: 4, 5, 6, 7
                                       (Requires Orthogonal DMRS Ports)
```

### 4.2 Frequency-Domain PRB Allocation Grid (1 Slot)

The rows below use a **single shared carrier and the same slot**. Read left to right as frequency increases: the overlap is the set of PRB indices where both UEs transmit simultaneously—not a separate frequency band between the cells.

```text
                         SAME CARRIER + SAME SLOT
Frequency increases  ----------------------------------------------------->

PRB index             0   1   2   3  |  4   5   6   7  |  8   9  10  11
                                      |<-- OVERLAP -->|
UE 1 (Cell A / TRP 1) [UE1 UE1 UE1 UE1]|[UE1 UE1 UE1 UE1]|[ .   .   .   . ]
UE 2 (Cell B / TRP 2) [ .   .   .   . ]|[UE2 UE2 UE2 UE2]|[UE2 UE2 UE2 UE2]
                                      |  same PRBs,     |
                                      |  same time      |

TRP 1 receives a mixture of UE1 + UE2 on PRBs 4-7; TRP 2 does too.
Both TRPs forward their received IQ to the centralized BBU/O-DU for joint
processing, which separates the streams using channel estimates and DMRS.
```

The detailed allocation per PRB is shown below. UE 1 uses PRBs 0–7; UE 2 uses PRBs 4–11, so PRBs 4–7 are allocated to both in the same slot.

```text
PRB Index |  Cell A Allocation (UE 1)  |  Cell B Allocation (UE 2)  |  UL CoMP Processing Mode
----------+----------------------------+----------------------------+-----------------------------------
  PRB 0   |  [      UE 1 PUSCH      ]  |  (        Unused        )  |  Single-Cell / Non-Overlapping
  PRB 1   |  [      UE 1 PUSCH      ]  |  (        Unused        )  |  Single-Cell / Non-Overlapping
  PRB 2   |  [      UE 1 PUSCH      ]  |  (        Unused        )  |  Single-Cell / Non-Overlapping
  PRB 3   |  [      UE 1 PUSCH      ]  |  (        Unused        )  |  Single-Cell / Non-Overlapping
==========+============================+============================+===================================
  PRB 4   |  [  UE 1 (DMRS Port 0)  ]  |  [  UE 2 (DMRS Port 1)  ]  |  >>> PRB OVERLAPPING REGION <<<
  PRB 5   |  [  UE 1 (DMRS Port 0)  ]  |  [  UE 2 (DMRS Port 1)  ]  |  Both TRPs receive UE 1 + UE 2.
  PRB 6   |  [  UE 1 (DMRS Port 0)  ]  |  [  UE 2 (DMRS Port 1)  ]  |  BBU separates streams via
  PRB 7   |  [  UE 1 (DMRS Port 0)  ]  |  [  UE 2 (DMRS Port 1)  ]  |  Joint MMSE-IRC & DMRS orthogonality
==========+============================+============================+===================================
  PRB 8   |  (        Unused        )  |  [      UE 2 PUSCH      ]  |  Single-Cell / Non-Overlapping
  PRB 9   |  (        Unused        )  |  [      UE 2 PUSCH      ]  |  Single-Cell / Non-Overlapping
  PRB 10  |  (        Unused        )  |  [      UE 2 PUSCH      ]  |  Single-Cell / Non-Overlapping
  PRB 11  |  (        Unused        )  |  [      UE 2 PUSCH      ]  |  Single-Cell / Non-Overlapping
```

### 4.3 Key Requirements in the Overlapping Region (PRBs 4–7)

* **Orthogonal DMRS Assignment:** Even though UE 1 connects to Cell A (PCI 101) and UE 2 connects to Cell B (PCI 102), the network must assign them **identical DMRS scrambling IDs (`n_SCID` and virtual cell ID)** while separating them via **orthogonal DMRS antenna ports** (e.g., Port 0 vs. Port 1, or different CDM groups). If each UE uses its own PCI to generate the DMRS sequence, cross-correlation interference corrupts channel estimation in PRBs 4–7.
* **Aligned Timing Advance (TA):** Both UE 1 and UE 2 must arrive at TRP 1 and TRP 2 within the same **Cyclic Prefix (CP)** window. If VIAVI's channel emulator introduces a large propagation delay delta between $\text{UE 1} \rightarrow \text{TRP 2}$ and $\text{UE 2} \rightarrow \text{TRP 1}$, orthogonality across PRBs 4–7 breaks down.
* **Balanced Receive Power (Rx PSD):** The gNB/eNB closed-loop power control must ensure the received power spectral density (PSD) per PRB of UE 1 and UE 2 at both TRPs falls within the receiver's dynamic range (typically within **10–15 dB** of each other) so the stronger UE does not saturate the ADC or drown out the weaker UE.

---

## 5. 5G NR FR1 Uplink CoMP Scenario (Band n78, 30 kHz SCS)

In **5G NR FR1 Uplink CoMP Joint Reception (Multi-TRP UL JR)**—typically deployed on mid-band frequencies such as **n78 (3.5 GHz) with 30 kHz Subcarrier Spacing (SCS)**—two cooperating gNB cells/TRPs receive simultaneous PUSCH transmissions from UEs located in their overlapping coverage area.

In 5G NR, there is no legacy LTE "CoMP mode" flag; instead, UL CoMP with PRB overlap is achieved using **Cross-Cell UL MU-MIMO** backed by **Virtual Cell ID (`dataScramblingIdentityPUSCH` and `scramblingID0`/`scramblingID1`)**, **Orthogonal DMRS Ports**, and **Centralized O-DU / BBU Joint IRC Processing**.

### 5.1 NR FR1 Multi-Cell / Multi-TRP Topology

In this example, **Cell 1 (PCI = 101)** and **Cell 2 (PCI = 102)** operate on the same NR-ARFCN carrier (e.g., 100 MHz bandwidth = 273 PRBs). **UE 1** (attached to Cell 1) and **UE 2** (attached to Cell 2) sit in the cell-edge overlap zone and are co-scheduled on **PRBs 50–99**.

```text
          [ gNB Cell 1 / TRP 1 ]                         [ gNB Cell 2 / TRP 2 ]
     (n78, 30 kHz SCS, PCI = 101)                   (n78, 30 kHz SCS, PCI = 102)

                   |                                              |
      Receives: y1 = h11*x1 + h12*x2                 Receives: y2 = h21*x1 + h22*x2

                   |                                              |
                   +---------------------\   /--------------------+
                                          \ /
                             ( O-RAN 7.2x Fronthaul / eCPRI )
                                           |
                                           v
                         +-----------------------------------+
                         |       Centralized O-DU / BBU      |
                         |   Joint UL MU-MIMO / MMSE-IRC     |
                         |  (Coherent Multi-TRP Combining)   |
                         +-----------------------------------+
                                    /                                                v               v
                           Decoded UE 1         Decoded UE 2
                          (PRBs 0 - 99)       (PRBs 50 - 149)

=====================================================================================
                      NR FR1 RF COVERAGE & OVERLAP ZONE (n78)

   ( . . . . . . Cell 1 Coverage . . . . . ( Overlap Zone ) . . . . . Cell 2 Coverage . . . . . )

                                           |              |
                                        [UE 1]         [UE 2]
                                    (PRBs 0 - 99)  (PRBs 50 - 149)

                                           |              |
                                           +------+-------+
                                                  |
                                     Overlapping PRBs: 50 to 99
                                (Shared Virtual DMRS Scrambling ID)
```

### 5.2 NR FR1 Frequency-Domain PRB Grid (1 Slot = 14 OFDM Symbols)

With **30 kHz SCS**, each slot is **0.5 ms** long, and the normal Cyclic Prefix (CP) is **~2.34 μs** (for symbols 1–13). The grid below shows how PRBs 50–99 overlap across both cells in the same slot.

```text
NR PRB Index |   Cell 1 Allocation (UE 1)    |   Cell 2 Allocation (UE 2)    |   O-DU Receiver Processing
-------------+-------------------------------+-------------------------------+-------------------------------------
 PRB 0 - 49  | [       UE 1 PUSCH          ] | (          Unused           ) | Single-TRP Reception (Cell 1)
=============+===============================+===============================+=====================================
             | [        UE 1 PUSCH         ] | [        UE 2 PUSCH         ] | >>> PRB OVERLAPPING REGION <<<
 PRB 50 - 99 | • DMRS Port: 1000 (CDM Grp 0) | • DMRS Port: 1002 (CDM Grp 1) | • Both TRPs forward IQ to O-DU
             | • Virtual DMRS ID: 500        | • Virtual DMRS ID: 500        | • Joint MMSE-IRC separates UE 1 & 2
             | • Data Scrambling ID: 500     | • Data Scrambling ID: 500     | • Requires < 2.34 μs delay delta
=============+===============================+===============================+=====================================
 PRB 100 - 149| (          Unused           ) | [       UE 2 PUSCH          ] | Single-TRP Reception (Cell 2)
```

### 5.3 Critical NR FR1 Parameters for VIAVI TM500 / E500 Verification

* **Virtual Cell ID Alignment (`scramblingID0` / `dataScramblingIdentityPUSCH`)**
  * **Why it matters:** By default, NR UEs initialize DMRS and PUSCH scrambling sequences using their serving cell's **PCI** (101 for UE 1, 102 for UE 2). In the overlapping region (PRBs 50–99), different PCIs break DMRS orthogonality.
  * **What to check in VIAVI RRC logs:** Confirm both UEs receive identical higher-layer scrambling IDs (e.g., `500`) in `DMRS-UplinkConfig` and `PUSCH-Config` via RRC Reconfiguration, and verify the TM500/E500 PHY layer applies them instead of falling back to PCI.
* **DMRS CDM Group Separation (`DCI Format 0_1` / `0_2`)**
  * **Why it matters:** In NR Type 1 DMRS, assigning **Port 1000 (CDM Group 0)** to UE 1 and **Port 1002 (CDM Group 1)** to UE 2 places their reference signals on completely separate subcarriers (even vs. odd) within PRBs 50–99, preventing DMRS pilot collisions even when propagation delays differ slightly.
* **30 kHz SCS Cyclic Prefix Window (~2.34 μs)**
  * **Why it matters:** Because NR FR1 mid-band uses 30 kHz SCS, the CP is half as long as LTE's (2.34 μs vs. 4.7 μs).
  * **What to check in VIAVI Channel Emulator:** The differential propagation delay between $\text{UE 1} \rightarrow \text{TRP 2}$ and $\text{UE 2} \rightarrow \text{TRP 1}$ (plus multipath delay spread) must stay strictly below **~2.3 μs** (equivalent to ~700 meters path difference). Exceeding this causes Inter-Symbol Interference (ISI) exclusively inside PRBs 50–99.
* **O-RAN 7.2x Fronthaul Section Handling (if testing O-DU)**
  * **Why it matters:** Over eCPRI fronthaul, Cell 1 sends U-Plane IQ data for PRBs 0–99 and Cell 2 sends PRBs 50–149. Ensure the VIAVI Fronthaul Tester / RU Emulator properly aligns `startPrbc` and `numPrbc` fields in eCPRI Section Type 1 packets so the O-DU equalizer aligns PRB 50 of Cell 1 with PRB 50 of Cell 2 in the same symbol buffer.
