# VIAVI UE Emulator Issues & Architecture Guide: Uplink CoMP Reception in PRB Overlapping Regions

## 1. Overview & Common Failure Symptoms

When testing **Uplink Coordinated Multi-Point (UL CoMP) Reception** (or Multi-TRP UL reception) with a **VIAVI UE Emulator** (such as the TM500 or E500), issues in a **Physical Resource Block (PRB) overlapping region** typically stem from synchronization mismatches, power control conflicts, DMRS (Demodulation Reference Signal) collisions, or fronthaul/channel model configuration errors.

**Issue statement:** PUSCH decoding may degrade when cooperating TRPs jointly receive a UE's uplink transmission, especially when other UEs share its PRBs. Check pilot separation, timing, frequency/phase tracking, receive power, and IQ transport before attributing the failure to PRB overlap itself.

Distinguish **one UE's same PUSCH received at multiple TRPs** from **different UEs sharing PRBs**. The first requires each receiver to process the same transmitted DMRS configuration; the second additionally requires coordinated pilot separation and sufficient spatial separability. These scenarios may exhibit:

* **High PUSCH BLER / CRC Failures:** The cooperating reception points fail to separate or coherently combine the overlapping uplink streams.
* **HARQ Retransmission Loops:** Excessive NACKs occur even when simulated path loss and SNR appear favorable.
* **UL Throughput Degradation:** The scheduler falls back to conservative Modulation and Coding Schemes (MCS) or drops out of CoMP Joint Reception (JR) mode into non-CoMP single-TRP reception.
* **DMRS Channel Estimation Errors:** Poor SINR reporting on the uplink DMRS, leading to inaccurate Interference Rejection Combining (IRC) or joint processing at the baseband unit (BBU/O-DU).

---

## 2. Primary Root Causes & Configuration Pitfalls

* **Timing Advance (TA) and Propagation Delay Mismatch**
  * **Cause:** In UL CoMP Joint Reception, signals from a UE arrive at different TRPs with different propagation delays. At each TRP, arrival-time spread and multipath must fit the receiver's CP/FFT-window budget. An inter-TRP delay difference alone does not establish a CP violation if receive windows are independently aligned.
  * **Impact:** An unsuitable receive window can cause ISI/inter-carrier interference (ICI), degrading channel estimation and joint reception, including on overlapping PRBs.
* **DMRS Port / Sequence Mismatch or Pilot Collisions**
  * **Cause:** When different UEs share PRBs in UL MU-MIMO / CoMP, the network coordinates DMRS mapping, ports, and sequences. Compatible orthogonal cover codes (OCC) on shared pilot REs or disjoint pilot REs can provide separation; different scrambling identities alone do not guarantee orthogonality. For NR with transform precoding disabled, `scramblingID0` / `scramblingID1` configure DMRS identities, while `n_SCID` selects the sequence initialization.
  * **Impact:** If the VIAVI UE script uses hardcoded parameters instead of following the applicable RRC/DCI configuration, or the receivers expect the wrong UE sequence, channel estimation and separation of overlapping streams can fail.
* **Residual Carrier-Frequency Offset / Phase-Tracking Error**
  * **Cause:** UE/receiver oscillator mismatch and Doppler produce frequency offsets that may differ by UE and receive point. Missing, stale, or inconsistent frequency-offset estimation (FOE) and compensation can leave phase evolution untracked during joint reception.
  * **Impact:** Residual offset degrades channel tracking and coherent combining; sufficiently large offsets also cause ICI. FOE is a receiver-PHY function, not a scheduler algorithm that is automatically disabled when CoMP becomes active (see Section 8).
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

* **Verify Channel Model Delays:** Check the VIAVI embedded fading/channel emulator settings between the UE and each TRP. At each receive point, verify FFT-window placement and arrival-time/multipath spread against the CP budget (approximately 4.7 μs for ordinary normal-CP symbols at 15 kHz SCS, or proportionally shorter for 30 kHz SCS). Independently aligned TRPs need not have identical propagation delays.
* **Inspect RRC & DCI Logs on the TM500/E500:**
  * Confirm that the UE is correctly parsing DCI Format 0_1 / 0_2 fields for **SRS Resource Indicator (SRI)** and **Antenna Ports / DMRS sequence initialization**.
  * For the same UE's PUSCH, verify that both receivers expect its actual DMRS ports, sequence, and mapping. For different UEs sharing PRBs, verify coordinated OCC or disjoint pilot REs; do not treat different scrambling IDs alone as proof of orthogonality.
* **Check FOE and Residual Phase:** Inspect available per-UE/per-TRP frequency-offset estimates, residual phase drift, and compensation settings. Compare single-TRP and joint-reception results with the same grants and channel conditions; a vendor-specific FOE switch requires its own documentation.
* **Check Uplink Power Headroom (PHR) & TPC:** Review the TM500/E500 PHY layer logs to confirm that UE transmit power is tracking the gNB's TPC commands and not hitting max power ($P_{\text{CMAX}}$) prematurely.
* **Isolate CoMP Modes:** Start with a non-overlapping PRB baseline, then test **Coordinated Scheduling/Beamforming (CS/CB)** and **Joint Reception (JR)** as supported. CS/CB does not inherently prohibit PRB reuse. This isolates basic multi-cell RF/timing synchronization from baseband IRC/equalizer issues.
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
                                       (Coordinated Pilot Separation)
```

### 4.2 Frequency-Domain PRB Allocation Grid (1 Slot)

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

For the port 0/1 example above, OCC separation requires compatible DMRS mapping and the same underlying sequence on the shared pilot REs, with sufficient channel coherence over the OCC span. Port numbers alone do not establish inter-UE pilot orthogonality.

### 4.3 Key Requirements in the Overlapping Region (PRBs 4–7)

* **Coordinated DMRS Assignment:** Different serving PCIs do not, by themselves, prevent joint reception. For shared-RE OCC separation (e.g., NR Type 1 ports 0 and 1), coordinate the underlying pilot sequence and mapping. Alternatively, Type 1 ports 0 and 2 use different CDM groups and disjoint pilot REs; protect both groups from the other UE's data. Identical DMRS identities across all UEs are **not a universal CoMP requirement**, and different sequences alone are not guaranteed orthogonal.
* **Aligned Timing Advance (TA):** At each TRP, the arrival-time spread of simultaneously processed UEs, including multipath, must fit the selected **CP/FFT window**. Check UE-to-TRP paths at each receiver rather than comparing unrelated paths across two receivers. Receive windows may be aligned independently before joint processing.
* **Balanced Receive Power (Rx PSD):** Keep each TRP's received PSD within its RF/ADC and equalizer operating range so the stronger UE does not overwhelm the weaker UE. A **10–15 dB** inter-UE difference can be an illustrative test target, not a universal receiver limit or 3GPP requirement.

### 4.4 Same UE at Serving and Cooperating TRPs: Which DMRS Port?

**Both TRPs receive the same DMRS port(s) and sequence transmitted in that UE's PUSCH. The UE does not obtain a separate DMRS port "from the cooperating cell" for combining.** For example, UE 1's port 0 is received as UE 1's port 0 at both TRPs; UE 2's pilot assignment is a separate scheduling decision.

A DMRS antenna port is a **logical transmit-side reference**, not a physical gNB receive antenna or connector. Each receive branch estimates its own channel for each transmitted port. The joint receiver uses these channel estimates and the received observations to separate users and combine the desired signal; matching port numbers alone does not perform combining.

This description applies to **one PUSCH transmission received simultaneously at multiple TRPs**. Separate PUSCH repetitions or independently configured multi-TRP transmissions must be checked against their own grants and configurations. See [3GPP TS 38.211, clauses 4.4.1 and 6.4.1.1](https://www.etsi.org/deliver/etsi_ts/138200_138299/138211/18.04.00_60/ts_138211v180400p.pdf).

---

## 5. 5G NR FR1 Uplink CoMP Scenario (Band n78, 30 kHz SCS)

In **5G NR FR1 Uplink CoMP Joint Reception (Multi-TRP UL JR)**—typically deployed on mid-band frequencies such as **n78 (3.5 GHz) with 30 kHz Subcarrier Spacing (SCS)**—two cooperating gNB cells/TRPs receive simultaneous PUSCH transmissions from UEs located in their overlapping coverage area.

In this 5G NR example, UL CoMP with PRB overlap uses **Cross-Cell UL MU-MIMO**, **Coordinated DMRS Configuration**, and **Centralized O-DU / BBU Joint Processing**. Joint reception is a network-side processing choice, not a requirement for every UE to share a "virtual cell ID." `dataScramblingIdentityPUSCH` controls data scrambling separately from DMRS `scramblingID0`/`scramblingID1`; identical data-scrambling identities across UEs are not required.

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
                                (Coordinated DMRS Configuration)
```

### 5.2 NR FR1 Frequency-Domain PRB Grid (1 Slot = 14 OFDM Symbols)

With **30 kHz SCS**, each slot is **0.5 ms** long, and the normal Cyclic Prefix (CP) is **~2.34 μs** (for symbols 1–13). The grid below shows how PRBs 50–99 overlap across both cells in the same slot.

```text
NR PRB Index |   Cell 1 Allocation (UE 1)    |   Cell 2 Allocation (UE 2)    |   O-DU Receiver Processing
-------------+-------------------------------+-------------------------------+-------------------------------------
 PRB 0 - 49  | [       UE 1 PUSCH          ] | (          Unused           ) | Single-TRP Reception (Cell 1)
=============+===============================+===============================+=====================================
             | [        UE 1 PUSCH         ] | [        UE 2 PUSCH         ] | >>> PRB OVERLAPPING REGION <<<
 PRB 50 - 99 | • DMRS Port: 0 (CDM Grp 0)    | • DMRS Port: 2 (CDM Grp 1)    | • Both TRPs forward IQ to O-DU
             | • Virtual DMRS ID: 500        | • Virtual DMRS ID: 500        | • Joint MMSE-IRC separates UE 1 & 2
             | • Data Scrambling ID: 500     | • Data Scrambling ID: 500     | • Per-TRP CP/FFT-window alignment
=============+===============================+===============================+=====================================
 PRB 100 - 149| (          Unused           ) | [       UE 2 PUSCH          ] | Single-TRP Reception (Cell 2)
```

The shared value `500` is an **example**, not a requirement for different UEs. The receivers must know each UE's actual configuration. For these Type 1 ports, reserve both CDM groups without data in each overlapping UE's allocation so one UE's data does not contaminate the other's pilots. NR **PUSCH** DMRS uses port numbering starting at **0**; the **1000-series** numbering applies to PDSCH DMRS.

### 5.3 Critical NR FR1 Parameters for VIAVI TM500 / E500 Verification

* **DMRS and Data Scrambling Configuration (`scramblingID0` / `scramblingID1` / `dataScramblingIdentityPUSCH`)**
  * **Why it matters:** For transform precoding disabled, the applicable RRC/DCI configuration determines DMRS identity and `n_SCID`, with PCI-based fallback where specified. PUSCH data scrambling is configured separately. Different serving PCIs do not automatically cause a pilot collision; pilot mapping, sequence alignment, and OCC determine separation.
  * **What to check in VIAVI RRC logs:** Verify each UE applies its configured identities and the applicable DCI sequence initialization, and that both TRPs process that UE's actual sequence. For shared-RE OCC multiplexing, also verify aligned underlying pilot sequences; for disjoint groups, verify protection from other-UE data. Do not force identical data-scrambling identities as a CoMP fix.
* **DMRS CDM Group Separation (`DCI Format 0_1` / `0_2`)**
  * **Why it matters:** In NR Type 1 DMRS, assigning **Port 0 (CDM Group 0)** to UE 1 and **Port 2 (CDM Group 1)** to UE 2 places their pilots on separate subcarrier combs (even vs. odd) within PRBs 50–99. Coordinate **CDM groups without data** across the UEs as well. This separates pilot REs but does not remove timing, frequency-offset, or spatial-separability requirements.
* **30 kHz SCS Cyclic Prefix Window (~2.34 μs)**
  * **Why it matters:** Because NR FR1 mid-band uses 30 kHz SCS, the CP is half as long as LTE's (2.34 μs vs. 4.7 μs).
  * **What to check in VIAVI Channel Emulator:** At each TRP, compare the co-scheduled UEs' arrival times and multipath spread with that receiver's FFT window and CP budget. An approximately **2.3 μs** budget corresponds to about **700 meters** of path-length difference, before other margins. This is not a universal limit on the same UE's inter-TRP propagation-delay difference when receive windows are independently aligned. CP violations can affect non-overlapping allocations too.
* **O-RAN 7.2x Fronthaul Section Handling (if testing O-DU)**
  * **Why it matters:** Over eCPRI fronthaul, Cell 1 sends U-Plane IQ data for PRBs 0–99 and Cell 2 sends PRBs 50–149. Ensure the VIAVI Fronthaul Tester / RU Emulator properly aligns `startPrbc` and `numPrbc` fields in eCPRI Section Type 1 packets so the O-DU equalizer aligns PRB 50 of Cell 1 with PRB 50 of Cell 2 in the same symbol buffer.

---

## 6. PRB Overlap Across Cells: Spectrum View

The PRBs are frequency positions on the **same carrier**. In the example below, both UEs transmit in the same slot, and PRBs 4–7 are assigned to both UEs. Each TRP therefore receives both signals on those PRBs; the centralized receiver uses channel estimates and DMRS to separate them.

```text
                         SAME CARRIER - SAME SLOT
Frequency increases  ----------------------------------------------------->

PRB index             0   1   2   3  |  4   5   6   7  |  8   9  10  11
                                      |<-- OVERLAP -->|
UE 1 (served by Cell A) [UE1 UE1 UE1 UE1]|[UE1 UE1 UE1 UE1]|[ .   .   .   . ]
UE 2 (served by Cell B) [ .   .   .   . ]|[UE2 UE2 UE2 UE2]|[UE2 UE2 UE2 UE2]
                                      |  Both transmit   |
                                      |  at same time    |

TRP 1 and TRP 2 receive UE 1 + UE 2 on PRBs 4–7.
They forward their received signals to the centralized BBU/O-DU for joint
processing.

```

## 7. VIAVI PHY & RRC Troubleshooting Matrix

Use the matrix below to map observed VIAVI TM500/E500 log counters and symptoms directly to root causes, 3GPP Information Elements (IEs), and corrective actions.

| VIAVI Log Symptom / Counter | Likely Root Cause | Target 3GPP IE / Parameter | Corrective Action |
| :--- | :--- | :--- | :--- |
| **PUSCH CRC Failure / High BLER** in allocations containing overlapping PRBs (e.g., PRBs 50–99), while a non-overlapping baseline passes. CRC is a transport-block result, not a per-PRB measurement. | Possible DMRS configuration mismatch, pilot/data collision, or insufficient spatial separation; different serving PCIs alone do not establish the cause. | `DMRS-UplinkConfig` (`scramblingID0`, `scramblingID1`), applicable DCI sequence initialization, and DMRS mapping | Match each UE's transmitted and expected sequence. Check shared-RE OCC compatibility or disjoint pilot REs protected from data; do not require a shared data-scrambling identity. |
| **Low DMRS SINR / Channel Estimation Error** despite high simulated SNR and correctly applied identities. | Pilot collision or incompatible OCC/sequence mapping; a matching identity alone does not establish orthogonality. | `DCI Format 0_1` / `0_2` (`Antenna ports` and CDM groups without data) & `dmrs-Type` | For Type 1, use ports `0` and `2` in CDM groups 0 and 1 with both groups protected from data, or validate shared-RE OCC separation. |
| **Inter-Symbol Interference (ISI) / Frequency-Dependent Phase Ramp** in 30 kHz SCS NR FR1 tests. | Incorrect receive timing can produce a phase ramp across subcarriers; exceeding the per-receiver CP budget can produce ISI/ICI. | VIAVI Channel Emulator Delay Profile, receiver FFT-window alignment & MAC `Timing Advance Command` | Check arrival-time spread and multipath at each TRP against its approximately 2.34 μs ordinary-symbol CP budget. Independently align receive windows; a phase ramp alone does not prove a CP violation. |
| **Phase Drift Across OFDM Symbols / Joint Reception Worse Than a Single TRP** despite correct DMRS mapping. | Residual frequency offset, stale channel/phase tracking, or inconsistent branch compensation. | Receiver FOE/CFO and phase-tracking diagnostics; vendor-specific controls, not a universal CoMP RRC IE | Check per-UE/per-TRP estimates and residual phase evolution. Keep compensation and channel estimates consistent; do not disable FOE solely because UL CoMP is active. |
| **Near-Far Capture Effect:** Stronger cell-edge UE decodes at 0% BLER while weaker co-scheduled UE hits ~100% BLER. | Uplink receive power imbalance exceeds the actual RF/ADC or MMSE-IRC operating range; no universal 15 dB failure threshold applies. | `PUSCH-PowerControl` ($P_0$, $\alpha$) & `DCI Format 0_1` (`TPC command`) | Adjust fractional path-loss compensation ($\alpha$) or closed-loop TPC targets against measured receiver limits. Treat 10–15 dB Rx PSD difference only as an illustrative test target. |
| **PHR Reports 0 dB / Negative Values** (`PHR <= 0 dB`) accompanied by rising BLER at cell edge. | UE hits maximum transmit power ($P_{\text{CMAX}}$) and cannot honor gNB closed-loop power-up TPC commands. | `p-Max` in `ServingCellConfigCommon` & VIAVI RF Tx power calibration | Reduce simulated path loss in the overlap zone or verify VIAVI RF front-end attenuation/gain calibration tables. |
| **eCPRI U-Plane Packet Drop / Truncated PRBs** in O-RAN 7.2x disaggregated test setups. | Misaligned PRB indexing or missing non-contiguous PRB support on the fronthaul interface. | eCPRI Section Type 1 (`startPrbc`, `numPrbc`) & VIAVI Fronthaul License | Verify `startPrbc` alignment across Cell 1 and Cell 2 IQ streams and ensure VIAVI non-contiguous PRB/multi-section modules are enabled. |
| **Scheduler Fails to Trigger UL CoMP JR** (keeps UEs on non-overlapping PRBs). | Stale or misaligned SRS CSI measurements preventing the gNB from pairing UEs spatially. | `SRS-Config` (`resourceMapping`, `transmissionComb`, `periodicityAndOffset`) | Align TM500 SRS comb offset and periodicity with the gNB's cross-cell SRS measurement window. |

---

## 8. FOE During Active UL CoMP: Receiver Role and Scheduler Impact

### 8.1 What FOE Estimates

**Frequency-offset estimation (FOE) estimates carrier-frequency offset (CFO); compensation and tracking use that estimate. These are receiver-PHY operations, even when a vendor exposes their controls through scheduler configuration.**

For UE $u$ and receive point $r$, an approximate offset model is:

$$
\Delta f_{u,r} = f_{\mathrm{UE},u} - f_{\mathrm{Rx},r} + f_{\mathrm{Doppler},u,r}.
$$

The corresponding phase evolves as $2\pi\Delta f_{u,r}t$. Independent receiver oscillators can create different offsets at different TRPs. A shared frequency reference reduces oscillator-related differences, but propagation-dependent Doppler may still differ. Receiver processing must account for the relevant per-UE/per-receive-point offsets; this does not require independent estimators on every antenna when branches share an oscillator.

Known pilots, including suitably separated DMRS observations when available, can support frequency/phase estimation. The applicable algorithm, estimation interval, and correction stage are implementation-specific; 3GPP does not mandate one universal gNB FOE algorithm. With multiple overlapping UEs, one frequency correction applied to the entire received mixture generally cannot perfectly compensate every UE.

### 8.2 Why FOE Still Matters When CoMP Is Active

* **Coherent IQ/symbol-level combining:** Each branch's channel and phase evolution must be tracked consistently. A static branch phase can be included in the channel estimate; untracked relative phase drift makes combining weights stale and reduces gain. Poorly weighted combining can underperform a good single branch.
* **Local demodulation followed by LLR combining:** Each receiver still needs adequate synchronization and demodulation. Cross-TRP RF-phase alignment is not required at the LLR-combining stage, but the soft information must correspond to the same coded transmission and be combined consistently.
* **Residual CFO:** Untracked offset causes phase error and degrades channel estimation; sufficiently large offsets also cause ICI. Using the same DMRS port at both TRPs does not remove these effects.

**There is no general rule to disable FOE because UL CoMP is active.** A vendor-specific "FOE enable/disable" switch may control a particular estimation or correction stage. Verify its documented meaning and avoid double-applying compensation or using channel estimates inconsistent with the corrected samples.

### 8.3 What the gNB Scheduler Sees

The scheduler selects grants, MCS, resources, and potentially cooperating users/points using available receiver measurements and decoding outcomes. It does **not** itself remove CFO from IQ samples.

| Receiver condition | Reception impact | Possible scheduler consequence |
| :--- | :--- | :--- |
| Accurate FOE with consistent compensation and channel tracking | Preserved coherent combining gain and more reliable UL quality measurements | More reliable link adaptation and potentially higher usable MCS |
| Residual CFO / phase-tracking error | Reduced effective SINR, higher BLER and retransmissions at a fixed MCS | Lower usable MCS, higher HARQ load, and lower throughput |
| Stale relative phase or an unreliable cooperating branch | Reduced joint-reception gain; joint reception may be worse than a good single branch | Implementation may change the cooperating set, user pairing, or reception mode |

These are **engineering consequences, not mandatory standardized scheduler reactions**. Accurate FOE alone does not guarantee a throughput gain if interference, timing, pilot collisions, or fronthaul errors remain.

### 8.4 Focused Verification

Compare serving-TRP-only, cooperating-TRP-only, and joint reception with the same PUSCH grants, channel conditions, and power settings. Where supported, inspect per-UE/per-TRP estimated CFO, residual phase drift over time, DMRS/channel-estimation quality, post-processing SINR, CRC/BLER, HARQ retransmissions, and throughput. Start at a fixed MCS to isolate reception changes before assessing adaptive scheduler behavior.

Do not infer CFO from a phase ramp across subcarriers alone: receive-timing error produces frequency-dependent phase, while CFO produces phase evolution over time and, when significant, ICI. No universal residual-CFO limit in Hz applies to every CoMP deployment; the tolerance depends on SCS, tracking interval, modulation, channel conditions, and receiver design.

## 9. Technical References and Validation Scope

* [3GPP TS 38.211 V18.4.0](https://www.etsi.org/deliver/etsi_ts/138200_138299/138211/18.04.00_60/ts_138211v180400p.pdf): clause 4.4.1 defines antenna ports; clause 6.3.1.1 covers PUSCH data scrambling; clause 6.4.1.1 covers PUSCH DMRS generation and mapping, including Type 1/2 port and CDM-group tables. Apply the clauses corresponding to the tested transform-precoding mode and release.
* [NVIDIA Sionna PUSCH DMRS configuration](https://github.com/NVlabs/sionna/blob/main/src/sionna/phy/nr/pusch_dmrs_config.py): an independent implementation cross-check for PUSCH port numbering, Type 1 CDM groups (ports 0/1 in group 0, ports 2/3 in group 1), OCC weights, scrambling identities, and CDM groups without data. This is supporting implementation evidence, not a normative specification.
* [An Analytical and Experimental Study of Distributed Uplink Beamforming in the Presence of Carrier Frequency Offsets](https://arxiv.org/abs/2508.08506): analytical and experimental evidence on distributed-receiver oscillator offsets, pilot-based channel/CFO measurements, and uplink SINR impact. This supports the receiver principles, not a specific VIAVI product feature or mandatory NR scheduler policy.

Vendor-specific FOE switches, counter names, receiver limits, and software/license capabilities require documentation for the exact gNB and emulator release. The scheduler effects and verification steps above are engineering guidance, not claims that a particular product implements a particular policy.
