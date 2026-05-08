# RF Fingerprinting Thesis — Experiment Repository

**Thesis:** Lightweight Device Authentication in Wireless Communication Using RF Fingerprinting  
**Authors:** Amitha Sanjaya & Tharangi Madushani  
**University:** Kristianstad University (HKR), DT339G VT26  
**Supervisor:** Prof. Qinghua Wang | **Examiner:** Ali Hassan Sodhro

---

## What This Is

This repo contains every experiment we ran for our thesis on lightweight RF fingerprinting for IoT device authentication. The idea: every radio transmitter has tiny manufacturing imperfections that make its signal unique. We capture those imperfections with a receiver and train a neural network to tell devices apart — no cryptographic keys, just hardware physics.

---

## Hardware

| Role | Device | Serial | Label |
|---|---|---|---|
| Transmitter 1 | USRP B200 | 3288FF2 | DEV01 (class 0) |
| Transmitter 2 | USRP B200 | 3467EEC | DEV02 (class 1) |
| Fixed Receiver | USRP B200 | 3288FAD | — |

**Recording settings:** 900 MHz · 1 MHz sample rate · TX/RX gain 30 · AGC off · FE corrections off

The receiver stays fixed throughout. Only the transmitter changes between device classes. This matters — if you swap the receiver too, you can't tell whether the classifier learned transmitter fingerprints or receiver differences.

---

## CNN Architecture

The same 43,874-parameter 1D CNN is used across all experiments:

```
Input (128, 2)          — 128 IQ samples, I and Q as two channels
  → Conv1D(32, k=7) + AveragePooling1D
  → Conv1D(64, k=5) + AveragePooling1D
  → Conv1D(128, k=3) + AveragePooling1D
  → GlobalAveragePooling1D
  → Dense(64, ReLU) + Dropout(0.5)
  → Dense(2, Softmax)
```

AveragePooling instead of Max (preserves subtle phase patterns), GlobalAveragePooling instead of Flatten (reduces params from ~500K to ~43K).

---

## Folder Structure

```
Session_01_Thesis_Analysis/
│
├── 📁 signal_analysis/              Step 1 — measure the hardware fingerprints
│   ├── README.md
│   ├── rf_fingerprint_full_analysis.ipynb      (Session 1, 14 sections)
│   └── rf_fingerprint_s2_v2_analysis.ipynb    (Session 2 + cross-session)
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
└── 📁 experiment_C_analysis/        Step 6 — one-class authenticator
    ├── README.md
    ├── experiment_C2_oneclass_embedding_svm.ipynb
    └── experiment_C3_DANN_oneclass.ipynb
```

---

## Run Order

If you want to reproduce everything from scratch, run in this order:

| Step | Folder | What to run | What it produces |
|---|---|---|---|
| 1 | `signal_analysis/` | S1 then S2 notebooks | CFO/RMS fingerprint measurements |
| 2 | `preprocessing/` | preprocessing notebook | 64 `.npy` files in `Preprocessed/` |
| 3 | `experiment_A_analysis/` | cnn_training → verification → documentation | `best_model_augmented.keras` |
| 4 | `experiment_B_analysis/` | B notebook | `B1_combined_model.keras` |
| 5 | `experiment_SQ3_analysis/` | SQ3 notebook | accuracy + latency comparison |
| 6 | `experiment_C_analysis/` | C2 then C3 notebooks | one-class authentication results |

Each notebook mounts Google Drive for data. Set `BASE = '/content/drive/MyDrive/My Thesis'` in Section 1 of each.

---

## Results Summary

| Experiment | Question | Key result |
|---|---|---|
| Signal Analysis | What hardware fingerprints exist? | CFO: 887 Hz (BPSK), 197 Hz (OOK) · RMS FDR: 22.8 (QPSK), 15.1 (GFSK) |
| Preprocessing | — | 64 `.npy` files, 3.29 GB, 100K windows/file |
| **Exp A** | SQ1 + SQ2: accuracy and SNR threshold | 99.98% at SNR0, FAR 0.00%, AUC 1.000 ✅ |
| **Exp B** | Multi-modulation | BPSK 100%, QPSK 97.85%, GFSK 98.36%, OOK 70.46% ✅ |
| **SQ3** | Raw IQ vs STFT spectrogram | Raw IQ wins on all metrics ✅ |
| **Exp C** | One-class authentication | BPSK: FAR 0.02% ✅ · Cross-modulation hits architectural limits |

---

## System Requirements — All Met

| Requirement | Target | Result |
|---|---|---|
| SR-1: Accuracy ≥ 95% at SNR ≥ 20 dB | ≥ 95% | 100% ✅ |
| SR-2: FAR ≤ 5% | ≤ 5% | 0.00% (Exp A + B) ✅ |
| PR-1: Latency < 10 ms | < 10 ms | 0.131 ms ✅ |
| PR-2: Model < 500K parameters | < 500K | 43,874 ✅ |
| PR-3: Accuracy ≥ 80% at SNR 10 dB | ≥ 80% | 100% ✅ |
| ER-4: Cross-session stability | Stable | 99.98% cross-session ✅ |

---

## Why Each Design Decision Was Made

Every choice eliminates one specific confound so what remains is hardware fingerprint only:

| Decision | What it removes |
|---|---|
| Fixed receiver | Receiver hardware effects |
| 4 modulations in training | Modulation-specific patterns |
| 2 sessions on different days | Session/noise floor memorisation |
| Noise augmentation at clean + SNR10 + SNR0 | Noise pattern memorisation |
| No mean subtraction | Accidental CFO removal |
| Max-amplitude normalisation only | Session-level power variation |
