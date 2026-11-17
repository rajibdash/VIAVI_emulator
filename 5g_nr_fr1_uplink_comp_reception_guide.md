# 5G NR FR1 Uplink CoMP Reception: A Standalone Guide

This guide explains how coordinated multi-point (CoMP) uplink reception works in 5G NR FR1, how it differs from other multi-TRP techniques, and what the UE, TRPs, and receiver need to do. It is an original explanation with self-contained diagrams; the diagrams are embedded below rather than linked to external image files.

The ShareTechnote CoMP page is included as further reading, not reproduced here. For conformance details, use the applicable 3GPP release and deployment profile.

## 1. The central idea

In ordinary single-point uplink reception, one serving TRP receives and decodes a UE's transmission. With uplink CoMP joint reception (JR), the UE sends an uplink transmission and two or more geographically separated reception points observe it. The network coordinates those observations to improve the chance of decoding the transmission.

The UE does not transmit a separate copy to every receiver merely because multiple TRPs listen. The same transmitted waveform can arrive at each TRP with a different channel, received power, delay, phase, and interference mixture. The network has to associate the observations with the same UE transmission and process them accordingly.

```text
                  One UE transmission
                 (PUSCH + its DMRS)
                          |
               +----------+----------+
               |                     |
               v                     v
         +-----------+         +-----------+
         | TRP A RX  |         | TRP B RX  |
         | yA = H_Ax |         | yB = H_Bx |
         +-----+-----+         +-----+-----+
               |                     |
               +----- received ------+
                     observations
                          |
                          v
               +---------------------+
               | Coordinated network |
               | receiver / decoder  |
               +----------+----------+
                          |
                  decoded UL block
```

`H_A` and `H_B` stand for the propagation and receiver effects at each point; `x` is the UE's transmitted signal. A second TRP is useful only when the network can make practical use of its observation. Merely enabling a second receiver does not guarantee a gain.

## 2. What “CoMP” can mean

CoMP describes coordination among transmission/reception points. It is an umbrella concept, not one specific receiver algorithm or one mandatory signaling procedure. In uplink, common engineering distinctions are:

| Approach | What is coordinated | What happens to the received signal |
|---|---|---|
| **Joint reception (JR)** | Multiple TRPs receive the same UE transmission and provide observations or derived information for joint decoding. | The receiver combines information across points before or during decoding. |
| **Selection / best-point reception** | The network chooses which TRP should provide the reception for a transmission. | One selected observation is decoded; this is not multi-branch combining. |
| **Coordinated scheduling (CS)** | Cells coordinate grants or user selection to reduce harmful interference. | Each TRP can still decode its own UE independently. |
| **Coordinated beamforming (CB)** | Scheduling and spatial transmission/reception choices are coordinated to manage interference. | Coordination does not itself imply that samples or decoded bits are jointly combined. |

These are useful conceptual categories; product implementations may use different names or combine functions. The important diagnostic question is what information crosses the TRP boundary: RF/IQ samples, symbols, soft bits, decoded blocks, or only scheduling and measurement information.

Downlink labels such as joint transmission and coordinated beamforming should not be mechanically applied to uplink. In uplink the UE is the transmitter and multiple network points are receivers.

## 3. FR1 and the resource grid

FR1 is the lower NR operating frequency range. The exact carrier, bandwidth, subcarrier spacing (SCS), cyclic prefix, and supported band depend on the configured NR band and release. A physical resource block (PRB) occupies 12 consecutive subcarriers for an OFDM symbol. Allocations are indexed in a cell's configured bandwidth part (BWP).

For joint reception over a shared carrier, the receive points must know how the UE's transmission maps to time-frequency resources. The same PRB index in two cells is useful for sample-wise or symbol-wise combining only when the receiver can relate the cells' carrier, numerology, timing, and resource-grid coordinates. Identical PRB numbers alone do not prove that the underlying resource elements are aligned.

```text
Frequency  --------------------------------------------------------------->

                   PRBs used by UE A
                 |-----------------------|
                              |-----------------------|
                              PRBs used by UE B
                              |<-- shared PRBs ------>|

Time       Slot n:    both grants overlap in OFDM-symbol time
                     and in the common frequency region

Interpretation: shared PRBs mean simultaneous resource reuse, not
automatic signal separation or guaranteed CoMP gain.
```

Two different cases are often confused:

1. **One UE, multiple receiving TRPs:** Each TRP observes the same UE's PUSCH. This is the basic uplink joint-reception case.
2. **Different UEs, reused PRBs:** Multiple UEs transmit on overlapping time-frequency resources. This is uplink multi-user multiplexing and requires the receiver to separate the users, using channel estimates, pilot design, spatial dimensions, and interference suppression. It is not implied by the first case.

Overlapping allocations may be intentional, but they can also be forbidden by a deployment's scheduler policy. CoMP does not make every PRB collision harmless.

## 4. Network architecture and information flow

The receiving points may belong to one gNB or to coordinated gNBs. A centralized or distributed baseband design can place the joint processing in different locations. The term “TRP” identifies a transmission/reception point; it does not by itself specify the CU/DU split, fronthaul interface, or receiver placement.

```text
                           Coordination
                  grants, UE context, timing
                     +------------------+
                     |                  |
                     v                  v
UE -- PUSCH --> [TRP A / RX]        [TRP B / RX] <-- PUSCH -- UE
                     |                  |
                     | UL observations  |
                     +--------+---------+
                              |
                 transport / internal interface
                              |
                              v
                  [Joint processing function]
                              |
                  decode, CRC, HARQ result
                              |
                              v
                    serving-cell protocol
```

The receiver interface and payload are implementation-specific. A joint processor might receive time-domain samples, frequency-domain samples, equalized symbols, or soft information. These choices have different transport-rate, synchronization, processing, and phase-coherence requirements. Do not assume that “forwarding UL data” necessarily means forwarding raw IQ.

The path also has a control side: the network needs consistent UE identity, cell/TRP association, slot and symbol timing, grant and HARQ context, and configuration for interpreting the received transmission. A correctly delivered data stream with stale or mismatched context can still fail to decode.

## 5. From a UE grant to a decoded transport block

An illustrative PUSCH reception sequence is:

```text
RRC configuration + UL grant
             |
             v
UE builds PUSCH and DMRS for the granted resources
             |
             v
TRPs receive their own noisy/channel-distorted observations
             |
             v
Each branch synchronizes and estimates its channel
             |
             v
Observations or soft information are aligned and combined
             |
             v
Demodulation -> de-rate matching/decoding -> CRC
             |
             +---- CRC pass: deliver block
             |
             +---- CRC fail: HARQ feedback / retransmission handling
```

This is a conceptual receiver chain. Real implementations can combine, reorder, or distribute the operations, and a standard does not mandate one universal gNB joint-reception algorithm.

### 5.1 PUSCH and DMRS

The physical uplink shared channel (PUSCH) carries uplink data and can also carry UCI as configured. Its demodulation reference signal (DMRS) provides a known pilot structure used for channel estimation and related receiver functions.

A **DMRS antenna port is a logical signal port**, not a physical receive antenna or an identifier for a receiving TRP. When multiple TRPs receive the same PUSCH, they estimate the channel from the DMRS that the UE actually transmitted. The UE does not normally choose a different DMRS port just because a second TRP is listening.

For different UEs sharing resources, the receiver must be able to distinguish the pilot observations sufficiently to estimate each user's channel. The applicable DMRS configuration, port, sequence, mapping, CDM group, and “CDM groups without data” behavior depend on the grant and RRC configuration. A port number by itself does not establish pilot orthogonality. Nor does assigning different scrambling identities by itself guarantee that two users' pilots will be orthogonal.

### 5.2 Channel estimation and combining

Each receive point sees a different effective channel. A receiver can use those channel estimates to weight the observations. With a simplified single-layer model:

```text
yA = hA x + nA
yB = hB x + nB

Combiner input: (yA, hA, noise/interference estimate A)
             + (yB, hB, noise/interference estimate B)
                              |
                              v
             estimate of the transmitted symbol x
```

The actual NR receiver may include multiple UE layers, receive antennas, interfering UEs, frequency-selective fading, and coding. Maximum-ratio combining, MMSE, interference-rejection combining (IRC), and soft-information combining are examples of receiver concepts—not a claim that every product uses a particular algorithm.

Combining is most effective when a branch is correctly synchronized, its channel/noise quality is estimated well, and its observation belongs to the same transmission. A badly aligned or incorrectly weighted branch can reduce performance; implementations may gate or exclude a poor branch.

## 6. Coherent and non-coherent forms of receiver cooperation

“Joint reception” does not always mean phase-coherent addition of raw RF samples.

| Processing point | What is combined | Main cross-TRP requirement |
|---|---|---|
| **Coherent sample/symbol combining** | Samples or complex symbols before decoding. | Accurate time/frequency mapping, per-branch channel and phase tracking; oscillator and transport effects must be handled. |
| **Soft combining** | Demodulator soft bits or log-likelihood ratios (LLRs). | Observations must refer to the same coded transmission and be calibrated/aligned in the decoder's representation. RF carrier phase coherence across sites is not necessarily required. |
| **Selection combining** | A choice of one branch or one decoded result. | Correct branch quality/CRC decisions; no joint phase combination. |

Coherent combining can exploit phase information but is more sensitive to residual timing, frequency, and phase errors. Soft combining can be more tolerant of independent RF phases, but still depends on valid channel estimation and consistent decoding context. These categories describe receiver engineering choices; their names and exact procedures vary.

## 7. Synchronization and radio conditions

### 7.1 Timing and cyclic prefix

For OFDM reception, each TRP needs an FFT window appropriate to the arrival-time and multipath spread it receives. The cyclic prefix (CP) provides tolerance to multipath delay under the receiver's timing assumptions. If the relevant spread exceeds the usable margin, inter-symbol and inter-carrier interference can degrade demodulation.

Do not treat the absolute propagation-delay difference between two sites as automatically a CP violation. The receiver can establish timing separately at each point, depending on architecture. What matters is the arrival-time spread and window placement for the signals that a particular receiver processes, plus whether the joint processor can associate corresponding symbols and samples across branches.

NR slot duration depends on numerology. For normal CP, a slot has 14 OFDM symbols; with SCS μ, the slot duration is 1 ms / 2^μ. The SCS and CP for the tested carrier must be read from its actual configuration rather than assumed from “FR1.”

### 7.2 Frequency and phase tracking

Residual carrier-frequency offset (CFO), Doppler, and phase noise cause phase evolution and can introduce inter-carrier interference. Different TRPs may have independent oscillators and propagation conditions. A coherent receiver therefore needs appropriate branch-specific synchronization and tracking, or another means of controlling relative phase.

Frequency-offset estimation (FOE) and compensation are receiver-PHY functions conceptually, even if a vendor exposes related controls through another configuration interface. CoMP activity is not a general reason to disable frequency tracking. Avoid double compensation and make sure channel estimates correspond to the samples being decoded.

### 7.3 Power and near-far effects

Each TRP has its own received-power balance. A strong UE can mask a weaker UE on shared PRBs or push an RF/ADC chain outside its useful dynamic range. Uplink power control, path loss, TPC commands, receiver gain, and interference estimates all affect decoding. A universal “safe” power difference cannot be stated without the receiver, bandwidth, waveform, and channel conditions.

### 7.4 Interference and spatial separability

Multiple receive antennas and geographically separated TRPs can provide additional observations, but the users still need sufficiently distinguishable channel signatures for multi-user separation. If users have highly correlated spatial channels, colliding pilots, or a severe power imbalance, adding another observation may not resolve the ambiguity.

## 8. NR multi-TRP terminology: avoid conflating procedures

NR includes standardized multi-TRP and uplink transmission procedures, with details evolving by release. Some procedures configure repeated PUSCH transmissions or multiple spatial relations/panels; those are not automatically identical to a network receiving one PUSCH simultaneously at multiple TRPs and jointly decoding its observations.

Keep the following questions separate:

* Did the UE transmit one PUSCH, or multiple repetitions/occasions?
* Did multiple TRPs receive the same transmission, or was one point selected per occasion?
* Does the network combine samples, symbols, soft bits, or decoded blocks?
* Is the objective receiver diversity, multi-user separation, interference coordination, or retransmission robustness?
* Which exact RRC/DCI procedure and 3GPP release configure the behavior?

The answers determine which DMRS, spatial-relation, repetition, HARQ, and receiver assumptions apply. “Multi-TRP” is not enough detail to infer them.

## 9. Expected benefits and practical tradeoffs

Potential benefits include improved coverage at the cell edge, receive diversity, improved robustness under fading, and the possibility of serving users whose signal is weak at one point but stronger at another. Uplink CoMP can also help a network exploit geographically distributed antennas.

Those benefits are conditional. Joint reception may require additional transport capacity, processing, buffering, scheduling coordination, and latency budget. If branches are poorly synchronized, have stale measurements, or experience common interference, the gain can be small or negative. CoMP also does not by itself increase the UE's transmit power or eliminate uplink interference.

The benefit should be measured against a controlled single-TRP baseline. Useful comparisons include block error rate (BLER), CRC success, post-processing SINR, HARQ retransmissions, throughput, and latency under the same grant, MCS, UE power, channel, and interference conditions.

## 10. A systematic verification workflow

1. **Establish the exact mode.** Record band, carrier, BWP, SCS, CP, cells/TRPs, release, whether the UE sends one PUSCH or repetitions, and where joint processing occurs.
2. **Capture the grant and UE configuration.** Verify slot/symbol allocation, PRBs, MCS, layers, DMRS configuration, applicable scrambling settings, and HARQ process. Check what each receiving point expects.
3. **Prove reception at each TRP separately.** Check signal presence, timing, frequency tracking, DMRS quality, channel estimate, receive power, and branch CRC or soft information before combining.
4. **Verify branch association and transport.** Confirm that the forwarded data are complete, timely, correctly labeled, and matched to the same UE, cell, slot, symbols, and HARQ context. Determine whether the payload is IQ, symbols, soft bits, or another representation.
5. **Compare one branch at a time.** Measure serving-TRP-only, cooperating-TRP-only, and joint reception with otherwise fixed conditions. If the joint result is worse than a good individual branch, inspect alignment, channel/noise weights, and context association.
6. **Test interference separately.** Begin with one UE and no co-scheduled interferer. Then add a second UE on disjoint resources, and finally test planned PRB reuse with coordinated pilots and grants. This separates basic joint reception from multi-user detection.
7. **Inspect HARQ and latency.** Confirm feedback timing, retransmission association, and processing/transport delay against the configured procedure. A successful decode at one point does not prove the combined result was delivered in time.
8. **Change one factor at a time.** Vary timing, power, pilot configuration, resource overlap, or participating TRP set independently and retain traces for the first failing slot.

### Symptom-to-investigation map

| Observation | First checks |
|---|---|
| Each TRP decodes, but joint reception fails | UE/slot/HARQ association, sample or soft-bit alignment, combining weights, duplicate/stale observations |
| One branch has poor DMRS quality | UE grant/configuration, pilot collisions, timing window, interference and branch RF health |
| Both branches degrade when a second UE is added | Pilot separation, spatial correlation, near-far balance, receiver MU-MIMO capability |
| IQ/sample combining is unstable but soft combining works | Relative timing/frequency/phase tracking, oscillator reference, fronthaul jitter and sample alignment |
| CRC succeeds but HARQ/reliability is poor | Feedback path, HARQ process mapping, timing, retransmission configuration |
| CoMP is enabled but no gain is measured | Confirm actual branch participation and processing mode; compare against a controlled baseline and verify the weak-branch quality |

## 11. Compact glossary

| Term | Meaning in this guide |
|---|---|
| **BWP** | Configured contiguous set of PRBs used for a UE's NR operation. |
| **CoMP** | Coordination among multiple transmission/reception points. |
| **CFO / FOE** | Carrier-frequency offset / its estimation. |
| **CP** | Cyclic prefix, the OFDM guard interval that supports multipath tolerance. |
| **DMRS** | Demodulation reference signal used for PUSCH channel estimation and related receiver tasks. |
| **FR1** | NR operating frequency range 1. |
| **HARQ** | Hybrid automatic repeat request, combining error control and retransmission. |
| **JR** | Joint reception of uplink observations from multiple points. |
| **LLR** | Log-likelihood ratio, a soft measure of coded-bit confidence. |
| **PUSCH** | Physical uplink shared channel. |
| **PRB** | Physical resource block: 12 consecutive subcarriers over an OFDM symbol. |
| **TRP** | Transmission/reception point; a radio point participating in network operation. |

## 12. References and further reading

* [ShareTechnote: 5G CoMP](https://www.sharetechnote.com/html/5G/5G_CoMP.html) — further reading on CoMP concepts. This document does not reproduce its text or illustrations.
* [3GPP TS 38.300, NR and NG-RAN overall description](https://www.3gpp.org/dynareport/38300.htm) — architecture and overall NR/NG-RAN description.
* [3GPP TS 38.211, NR physical channels and modulation](https://www.3gpp.org/dynareport/38211.htm) — physical signals/channels, resource mapping, and reference signals.
* [3GPP TS 38.213, NR physical layer procedures for control](https://www.3gpp.org/dynareport/38213.htm) and [TS 38.214, NR physical layer procedures for data](https://www.3gpp.org/dynareport/38214.htm) — procedures related to physical-layer control and data.
* [3GPP TS 38.331, NR RRC protocol specification](https://www.3gpp.org/dynareport/38331.htm) — RRC configuration.

Use the release/version corresponding to the network and UE under test. The conceptual diagrams above are original illustrations, not copies of external images.
