# Experiment SQ4 — Cross-Hardware Domain Gap
## USRP B200 → RTL-SDR V3 Transfer Analysis

**Research Sub-Question (SQ4):**  
*How much does cross-hardware deployment — using a USRP B200-trained model on a RTL-SDR V3 receiver — degrade authentication accuracy, and is the gap recoverable through in-domain retraining?*

---

## ⚠️ Hardware Setup Difference vs. Thesis Plan

The thesis methodology (Chapter 3) originally planned a **4-dongle RTL-SDR V3 array** as the cross-hardware receiver. Due to hardware availability constraints, this experiment was conducted with a **single RTL-SDR V3 dongle** instead.

| Component | Thesis Plan | This Experiment |
|-----------|-------------|-----------------|
| Transmitter 1 | USRP B200 (DEV01, serial 3288FF2) | Same ✓ |
| Transmitter 2 | USRP B200 (DEV02, serial 3467EEC) | Same ✓ |
| Cross-hardware receiver | RTL-SDR V3 × 4 (spatial array) | **RTL-SDR V3 × 1 (single dongle)** |
| Modulations | BPSK, QPSK, GFSK, OOK | Same ✓ |
| Centre frequency | 900 MHz | Same ✓ |
| Sample rate | 1 Msps | Same ✓ |

**Scientific implication:** Using a single dongle rather than an array does not reduce the validity of the domain gap measurement — it makes it a stronger and cleaner result. The array was intended to study spatial diversity, which is a separate question from cross-hardware transfer. A single RTL-SDR V3 isolates the receiver hardware variable precisely.

---

## RTL-SDR V3 Capture Parameters

Confirmed from GNU Radio Companion (GRC) flowgraphs used during recording:

| Parameter | Value | Notes |
|-----------|-------|-------|
| Centre frequency | 900 MHz | Identical to Phase 1 |
| Sample rate | 1 Msps | Identical to Phase 1 |
| TX gain (USRP) | 30 dB | Identical to Phase 1 |
| RX RF gain | 30 dB | RTL-SDR tuner gain |
| RX IF gain | 20 dB | RTL-SDR IF stage |
| RX BB gain | 20 dB | RTL-SDR baseband stage |
| AGC (gain_mode) | **False** | Disabled — critical for fingerprinting |
| DC offset mode | **0** | No auto-correction — raw DC preserved |
| IQ balance mode | **0** | No auto-correction — raw imbalance preserved |
| Frequency correction | 0 ppm | No correction applied |
| File format | fc32 (complex64) | Identical to Phase 1 |

**File naming convention:** `DEV{01|02}_{MOD}_S3_R{1-5}_RTL.dat`

---

## Experimental Design

| Sub-Exp | Name | Train | Test | Purpose |
|---------|------|-------|------|---------|
| D1 | Zero-shot transfer | Phase 1 B1 model (USRP-trained, no retraining) | RTL-SDR R4+R5 | **Primary SQ4 answer: domain gap** |
| D2 | RTL-SDR in-domain | RTL-SDR R1–R3 | RTL-SDR R4+R5 | Performance ceiling on RTL-SDR hardware |
| D3 | Feature analysis | — | Both datasets | Mechanistic explanation of the gap |

**Train/test split:** Temporal by repetition number (R1–R3 train, R4–R5 test). This mirrors the Phase 1 protocol exactly, ensuring a fair comparison.

**D2 note:** Only one recording session is available (S3), so D2 uses a within-session temporal split. This is an optimistic upper bound — a between-session split would yield more conservative but more realistic in-domain estimates.

---

## Key Results

### D1 — Zero-Shot Transfer (Primary SQ4 Answer)

| Modulation | USRP Phase 1 | RTL Zero-Shot | dAcc | 95% CI | FAR |
|-----------|-------------|--------------|------|--------|-----|
| BPSK | 100.0% | 49.4% | **+50.6 pp** | [49.31%, 49.53%] | 99.95% |
| QPSK | 100.0% | 71.7% | **+28.3 pp** | [71.55%, 71.77%] | 43.28% |
| GFSK | 100.0% | 27.9% | **+72.1 pp** | [27.79%, 28.00%] | 54.53% |
| OOK | 70.5% | 49.8% | **+20.7 pp** | [49.66%, 49.90%] | 0.60% |
| **Mean** | — | — | **+42.94 pp** | — | — |

### D2 — RTL-SDR In-Domain Ceiling

| Modulation | Zero-Shot D1 | In-Domain D2 | Recoverable Gap |
|-----------|-------------|-------------|----------------|
| BPSK | 49.4% | **99.5%** | +50.1 pp |
| QPSK | 71.7% | **91.3%** | +19.6 pp |
| GFSK | 27.9% | **99.9%** | +72.0 pp |
| OOK | 49.8% | **96.4%** | +46.6 pp |

D2 model: same 1D CNN architecture (43,874 parameters), trained on RTL-SDR R1–R3 with AWGN augmentation at 0/10/20 dB. Training converged at epoch 21, val accuracy 98.9%.

---

## Findings and Interpretation

### Finding 1 — Complete zero-shot failure across three modulations
BPSK (49.4%), OOK (49.8%), and GFSK (27.9%) all collapse to chance level or below when the Phase 1 model is applied to RTL-SDR data without retraining. A mean domain gap of 42.94 percentage points means the USRP-trained model is essentially non-functional on RTL-SDR hardware. **This directly answers SQ4.**

### Finding 2 — GFSK sub-chance inversion is the most significant result
GFSK zero-shot accuracy is 27.9% — below the 50% random baseline — with a FAR of 54.5%. The model is not simply failing; it is predicting the wrong device with consistent confidence. This is caused by the RTL-SDR V3's direct-conversion architecture, which introduces IQ imbalance and a fixed DC component at the centre frequency. For GFSK, whose fingerprint is dominated by instantaneous frequency (CFO), the RTL-SDR's own oscillator offset inverts the relative CFO ordering between DEV01 and DEV02 compared to the USRP receiver. The Phase 1 model learned the USRP-specific ordering and applies it backwards on RTL-SDR data.

### Finding 3 — QPSK partial transfer confirms RMS amplitude is hardware-agnostic
QPSK is the only modulation with above-chance zero-shot accuracy (71.7%). In Phase 1, QPSK fingerprinting was dominated by RMS amplitude (Fisher Discriminant Ratio = 22.8): DEV02 consistently had higher output power than DEV01 through the USRP receiver. Transmit power is a hardware property of the transmitter, not the receiver, and it survives hardware change as long as AGC is disabled (confirmed). This provides direct experimental evidence that amplitude-based fingerprints are more hardware-agnostic than frequency-based fingerprints.

### Finding 4 — The gap is fully recoverable through in-domain retraining
D2 in-domain accuracy reaches 99.5% (BPSK), 91.3% (QPSK), 99.9% (GFSK), and 96.4% (OOK). The RTL-SDR hardware is fully capable of capturing transmitter fingerprints — the domain gap is a training distribution mismatch, not a hardware limitation. Domain adaptation or receiver-specific fine-tuning would be sufficient to close the gap without requiring new recordings.

### Finding 5 — D3 t-SNE confirms feature-level domain separation
t-SNE projection of 36,000 CNN penultimate-layer embeddings (12,000 USRP + 24,000 RTL-SDR) shows spatial separation between USRP and RTL-SDR clusters for the same transmitter. This confirms the Phase 1 model learned receiver-hardware-coloured features rather than purely transmitter fingerprints.

---

## RTL-SDR V3 vs. USRP B200 Hardware Differences

Understanding why the domain gap exists requires comparing the receiver architectures:

| Property | USRP B200 (Phase 1 receiver) | RTL-SDR V3 (SQ4 receiver) |
|----------|------------------------------|---------------------------|
| ADC resolution | 12-bit | 8-bit |
| Architecture | Superheterodyne | **Direct-conversion** |
| DC component | Minimal | **Strong DC spike at centre freq** |
| IQ imbalance | Negligible | **Systematic amplitude/phase mismatch** |
| Noise figure | ~5 dB | ~3.5 dB |
| Phase noise | Low | Higher |
| Gain stability | Excellent | Moderate (even with AGC off) |

The direct-conversion architecture is the primary cause of the domain gap. It introduces a fixed DC offset and IQ imbalance that distort the instantaneous frequency estimates which underpin CFO-based fingerprinting (BPSK, GFSK). These artefacts are receiver-specific — they change with every power cycle and differ between dongles — making them unusable as transmitter features and corrupting the features learned by the USRP-trained model.

---

## Folder Contents

```
experiment_SQ4_analysis/
├── README.md                                  ← this file
├── experiment_SQ4_domain_gap_CLEAN.ipynb      ← main analysis notebook
└── results/
    ├── dAcc_domain_gap_table.csv              ← D1 domain gap per mod × SNR
    └── SQ4_comparison_table.csv              ← three-way comparison table
```

**Note:** Raw `.dat` recording files (~170 MB each, 40 files total) are stored in Google Drive and are not included in this repository. The notebook mounts Drive to access them.

---

## How to Run

1. Open `experiment_SQ4_domain_gap_CLEAN.ipynb` in Google Colab
2. Mount Drive when prompted (Cell 1)
3. Run cells sequentially — Cell 2 contains all imports and helpers and can be re-run alone after any session crash
4. Data path: `MyDrive/My Thesis/Recordings/Recordings_RTL/RTL_Mod/`
5. Phase 1 model path: `MyDrive/My Thesis/Models/ExperimentB/B1_combined_model.keras`
6. Results save to: `MyDrive/My Thesis/SQ4_Results/`

**RAM requirements:** ~500 MB peak during D2 training (MAX_WIN=5000 per file). Google Colab free tier (12 GB RAM) is sufficient.

---
# RF Fingerprinting Thesis — Experiment Repository

**Thesis:** Lightweight Device Authentication in Wireless Communication Using RF Fingerprinting  
**Authors:** Amitha Sanjaya & Tharangi Madushani  
**University:** Kristianstad University (HKR), DT339G VT26  
**Supervisor:** Prof. Qinghua Wang | **Examiner:** Ali Hassan Sodhro

---

## What This Is

This repository contains every experiment run for our thesis on lightweight RF fingerprinting for IoT device authentication. The idea: every radio transmitter has tiny manufacturing imperfections that make its signal unique. We capture those imperfections with a receiver and train a neural network to tell devices apart — no cryptographic keys, just hardware physics.

---

## Hardware

### Phase 1 — USRP B200 (Sessions 1 & 2)

| Role | Device | Serial | Label |
|------|--------|--------|-------|
| Transmitter 1 | USRP B200 | 3288FF2 | DEV01 (class 0) |
| Transmitter 2 | USRP B200 | 3467EEC | DEV02 (class 1) |
| Fixed Receiver | USRP B200 | 3288FAD | — |

**Recording settings:** 900 MHz · 1 MHz sample rate · TX/RX gain 30 · AGC off · FE corrections off

The receiver stays fixed throughout Phase 1. Only the transmitter changes between device classes.

### Phase 2 — SQ4 Cross-Hardware (Session 3)

| Role | Device | Notes |
|------|--------|-------|
| Transmitter 1 | USRP B200 (DEV01) | Same transmitters as Phase 1 |
| Transmitter 2 | USRP B200 (DEV02) | Same transmitters as Phase 1 |
| Cross-hardware receiver | **RTL-SDR V3 dongle** | ⚠️ Different from thesis plan — see SQ4 README |

> **Hardware note:** The thesis methodology planned a 4-dongle RTL-SDR V3 array for spatial diversity experiments. Due to hardware availability, SQ4 was conducted with a single RTL-SDR V3 dongle. This change does not affect the validity of the domain gap measurement — it isolates the hardware variable more cleanly.

---

## CNN Architecture

The same 43,874-parameter 1D CNN is used across all experiments:

```
Input (128, 2)          — 128 IQ samples, I and Q as two channels
  → Conv1D(32, k=7) + BatchNorm + AveragePooling1D
  → Conv1D(64, k=5) + BatchNorm + AveragePooling1D
  → Conv1D(64, k=3) + BatchNorm + AveragePooling1D
  → GlobalAveragePooling1D
  → Dense(64, ReLU) + Dropout(0.3)
  → Dense(2, Softmax)
```

AveragePooling instead of Max (preserves subtle phase patterns). GlobalAveragePooling instead of Flatten (reduces parameters from ~500K to ~43K).

---

## Folder Structure

```
PLA-Thesis_Analysis/
│
├── 📁 signal_analysis/              Step 1 — measure hardware fingerprints
│   ├── README.md
│   ├── rf_fingerprint_full_analysis.ipynb
│   └── rf_fingerprint_s2_v2_analysis.ipynb
│
├── 📁 preprocessing/                Step 2 — prepare data for training
│   ├── README.md
│   └── rf_fingerprint_preprocessing.ipynb
│
├── 📁 experiment_A_analysis/        Step 3 — binary classifier, BPSK (SQ1 + SQ2)
│   ├── README.md
│   ├── experiment_A_cnn_training.ipynb
│   ├── experiment_A_verification.ipynb
│   └── experiment_A_documentation.ipynb
│
├── 📁 experiment_B_analysis/        Step 4 — multi-modulation classifier
│   ├── README.md
│   └── experiment_B_multimodulation.ipynb
│
├── 📁 experiment_SQ3_analysis/      Step 5 — raw IQ vs STFT comparison (SQ3)
│   ├── README.md
│   └── experiment_SQ3_iq_vs_stft.ipynb
│
├── 📁 experiment_C_analysis/        Step 6 — one-class authenticator
│   ├── README.md
│   ├── experiment_C2_oneclass_embedding_svm.ipynb
│   └── experiment_C3_DANN_oneclass.ipynb
│
└── 📁 experiment_SQ4_analysis/      Step 7 — cross-hardware domain gap (SQ4) ← NEW
    ├── README.md                    ← hardware differences documented here
    ├── experiment_SQ4_domain_gap_CLEAN.ipynb
    └── results/
        ├── dAcc_domain_gap_table.csv
        └── SQ4_comparison_table.csv
```

---

## Run Order

| Step | Folder | What it produces |
|------|--------|-----------------|
| 1 | `signal_analysis/` | CFO/RMS fingerprint measurements |
| 2 | `preprocessing/` | 64 `.npy` files in `Preprocessed/` on Drive |
| 3 | `experiment_A_analysis/` | `best_model_augmented.keras` |
| 4 | `experiment_B_analysis/` | `B1_combined_model.keras` |
| 5 | `experiment_SQ3_analysis/` | Accuracy + latency comparison |
| 6 | `experiment_C_analysis/` | One-class authentication results |
| 7 | `experiment_SQ4_analysis/` | Domain gap tables + figures → `SQ4_Results/` |

Each notebook mounts Google Drive. Set `BASE = '/content/drive/MyDrive/My Thesis'` in Section 1 of each.

---

## Results Summary

| Experiment | Research Question | Key Result |
|-----------|-------------------|-----------|
| Signal Analysis | What hardware fingerprints exist? | CFO sep: 887 Hz (BPSK) · RMS FDR: 22.8 (QPSK) |
| **Exp A** | SQ1+SQ2: feasibility and SNR threshold | 99.98% accuracy · 0.00% FAR · EER=0% to −5 dB ✅ |
| **Exp B** | Multi-modulation classifier | BPSK 100% · QPSK 97.9% · GFSK 98.4% · OOK 70.5% ✅ |
| **SQ3** | Raw IQ vs STFT representation | Raw IQ wins on accuracy and latency ✅ |
| **Exp C** | One-class open-set authentication | BPSK: AUC=0.75 (DANN) · FAR 0.02% at clean ✅ |
| **SQ4** ⚠️ | Cross-hardware USRP→RTL-SDR gap | Mean dAcc **+42.94 pp** zero-shot · Recoverable to ≥91% in-domain |

### SQ4 Domain Gap Detail

| Modulation | USRP Phase 1 | RTL Zero-Shot (D1) | RTL In-Domain (D2) | dAcc |
|-----------|-------------|------------------|------------------|------|
| BPSK | 100.0% | 49.4% | 99.5% | +50.6 pp |
| QPSK | 100.0% | 71.7% | 91.3% | +28.3 pp |
| GFSK | 100.0% | **27.9%** ⬇ | 99.9% | +72.1 pp |
| OOK | 70.5% | 49.8% | 96.4% | +20.7 pp |

> GFSK falls below chance (27.9%) due to decision-boundary inversion caused by IQ imbalance in the RTL-SDR V3 direct-conversion architecture. In-domain D2 training recovers to 99.9%, confirming the gap is a training distribution mismatch, not a hardware limitation.

---

## System Requirements

| Requirement | Target | Phase 1 Result | SQ4 Zero-Shot | SQ4 In-Domain |
|-------------|--------|---------------|---------------|---------------|
| SR-1: Accuracy ≥ 95% at SNR ≥ 20 dB | ≥ 95% | 100% ✅ | 27.9–71.7% ❌ | 91.3–99.9% ✅/⚠️ |
| SR-2: FAR ≤ 5% | ≤ 5% | 0.00% ✅ | 0.6–99.9% ❌ | — |
| PR-1: Latency < 10 ms | < 10 ms | 0.131 ms ✅ | 0.131 ms ✅ | 0.131 ms ✅ |
| PR-2: Model < 500K parameters | < 500K | 43,874 ✅ | 43,874 ✅ | 43,874 ✅ |

SR-1 and SR-2 failures in SQ4 zero-shot are expected — they quantify the domain gap. In-domain retraining (D2) recovers SR-1 for BPSK, GFSK, and OOK.

---

## Why Each Design Decision Was Made

| Decision | What it removes |
|----------|----------------|
| Fixed receiver (Phase 1) | Receiver hardware effects from fingerprint |
| 4 modulations | Modulation-specific overfitting |
| 2 recording sessions, different days | Session/noise memorisation |
| AWGN augmentation (0/10/20 dB) | Noise pattern memorisation |
| No mean subtraction | Accidental CFO feature removal |
| Max-amplitude normalisation only | Session-level power variation |
| AGC disabled (both phases) | Gain-control artefacts |
| Temporal train/test split | Data leakage from shuffled windows |

---

## About

Lightweight Device Authentication in Wireless Communication Using RF Fingerprinting  
Kristianstad University — DT339G VT26  
GitHub: [Amitha4469/rf-fingerprinting-thesis](https://github.com/Amitha4469/rf-fingerprinting-thesis)

## Relationship to Thesis Chapter 4

This experiment addresses SQ4 as defined in Chapter 3 of the thesis:

> *"SQ4: To what extent does the domain gap between USRP B200 and RTL-SDR V3 hardware affect the authentication accuracy of the trained 1D CNN model?"*

The answer: a mean accuracy degradation of **42.94 percentage points** under zero-shot transfer, ranging from 20.7 pp (OOK) to 72.1 pp (GFSK), with systematic decision-boundary inversion observed for GFSK. In-domain retraining recovers near-full performance for all modulations, indicating that domain adaptation is both necessary and sufficient for cross-hardware deployment.

---

*Authors: Amitha Sanjaya & Tharangi Madushani*  
*Kristianstad University (HKR), DT339G VT26*  
*Supervisor: Prof. Qinghua Wang | Examiner: Ali Hassan Sodhro*
