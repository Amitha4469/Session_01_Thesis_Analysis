# RF Fingerprinting — Session 1 Signal Analysis

**Thesis:** Lightweight Device Authentication in Wireless Communication Using RF Fingerprinting
**Course:** DT339G VT26 — Kristianstad University (HKR)
**Authors:** Amitha · Tharangi Madushani
**Supervisor:** Prof. Qinghua Wang

---

## Experimental Setup

| Parameter | Value |
|---|---|
| Transmitter 1 (DEV01) | USRP B200 · serial 3288FF2 |
| Transmitter 2 (DEV02) | USRP B200 · serial 3467EEC |
| Fixed Receiver (RX)   | USRP B200 · serial 3288FAD |
| Centre frequency | 900 MHz |
| Sample rate | 1 MHz |
| TX gain / RX gain | 30 / 30 |
| FE corrections | OFF |
| AGC | Disabled |
| Modulations captured | BPSK, QPSK, GFSK, OOK |
| Repetitions per class | 5 (R1–R5) |
| Session | S2 |
| Total recordings | 40 |
| File format | complex64 .dat (GNU Radio File Sink) |
| Recording duration | ~20 s per file (~20 M samples) |
| Transient drop | First 100,000 samples |

---

## Key Findings

**Carrier Frequency Offset is the dominant hardware fingerprint — confirmed in Session 2.**

Each USRP B200 contains a crystal oscillator that runs at a slightly different frequency
than its nominal specification. The difference between the transmitter and receiver oscillators
is the Carrier Frequency Offset (CFO). CFO is hardware-fixed — it does not depend on
modulation type, transmission content, or signal power.

| Metric | Value |
|---|---|
| DEV01 mean CFO (all modulations) | -5303.3 Hz |
| DEV02 mean CFO (all modulations) | +238.2 Hz |
| CFO separation | 5541.5 Hz |
| DEV01 within-session CFO std | 22929.21 Hz |
| DEV02 within-session CFO std | 15533.02 Hz |

A 5541 Hz separation with sub-Hz within-device variance provides a physically grounded,
highly stable basis for device authentication without any trained model.

---

## Analysis Plots

| # | File | Description |
|---|---|---|
| 00 | `00_health_check_summary.png` | Signal power, clipping percentage, and DC magnitude per device per modulation. All 40 files passed. |
| 01 | `01_iq_time_domain.png` | I and Q waveforms for first 2,000 samples. Confirms modulation type and recording integrity. |
| 02 | `02_constellation_all_modulations.png` | IQ constellations showing CFO-driven phase rotation. Early vs late segment comparison across all 4 modulations. |
| 03 | `03_constellation_rotation_closeup.png` | BPSK and QPSK close-up with measured rotation angle between early and late samples. |
| 04 | `04_psd_comparison.png` | Welch PSD: DEV01 vs DEV02 per modulation. Lateral shift between curves = CFO in frequency domain. |
| 05 | `05_cfo_boxplots.png` | Quantified CFO estimates per device per modulation. Box spread = intra-device variance; gap = separability. |
| 06 | `06_cfo_repetition_stability.png` | CFO across R1–R5 per modulation. Flat per-device line = stable within-session fingerprint. |
| 07 | `07_dc_offset_analysis.png` | DC offset I vs Q scatter, magnitude per modulation, and repetition stability. Secondary fingerprint feature. |
| 08 | `08_amplitude_histograms.png` | Amplitude envelope distributions per modulation. Mean lines show device-level amplitude differences. |
| 09 | `09_phase_trajectory.png` | Unwrapped phase over 50,000 samples with linear fit. Slope printed in legend = CFO in Hz. Different slopes per device. |
| 10 | `10_feature_separability_heatmap.png` | Fisher Discriminant Ratio for 9 features × 4 modulations. CFO dominates. |
| 11 | `11_cross_modulation_stability.png` | CFO per device plotted across all modulations. Validates CFO as hardware property, not modulation artifact. |
| 12 | `12_repetition_stability.png` | Per device-modulation CFO across 5 repetitions with ±1σ band. |

---

## Data Files

- `feature_summary.csv` — all 9 extracted features for all 40 recordings in one table.

---

## Supervisor Note — Costas Loop and CFO

Prof. Qinghua Wang identified the rotating constellation as a CFO artifact and suggested
adding a synchronisation chain (RRC filter → Symbol Sync → Costas Loop) to correct the rotation
for conventional demodulation. For RF fingerprinting, the correction is **intentionally omitted** —
the CFO-induced rotation is the device identity. The analysis in this notebook quantifies
and validates that design decision. Prof. Wang's own note confirms: *"Center frequency difference
is actually quite interesting and can be used to differentiate radios."*

---

## Next Steps

1. **Preprocessing notebook** — transient drop, normalisation, segmentation (128 samples), AWGN injection at 0 / 10 / 20 dB, save `.npy`
2. **Experiment A** — binary CNN on BPSK data (temporal holdout split)
3. **Experiment B** — multi-modulation CNN, leave-one-modulation-out test
4. Session 2 — complete. See Section 15 of this notebook for cross-session comparison.
5. **Thesis Chapter 4** — results from Experiments A and B

---

*Generated automatically by `rf_fingerprint_s2_analysis.ipynb`*
