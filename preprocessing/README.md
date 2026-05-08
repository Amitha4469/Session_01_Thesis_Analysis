# Preprocessing Pipeline

**Notebook:** `rf_fingerprint_preprocessing.ipynb`  
**Output:** 64 `.npy` files, 3.29 GB total → `Preprocessed/S1/` and `S2/`

---

## What This Does

Takes the raw `.dat` recordings from GNU Radio and converts them into clean, windowed, noise-augmented numpy arrays ready for CNN training.

---

## Pipeline Steps

### Step 1 — Drop the transient (first 100,000 samples)

When a USRP powers on and starts transmitting, the hardware takes a moment to stabilise. The first ~100,000 samples show a power-on surge that doesn't represent steady-state hardware behaviour. We drop them.

### Step 2 — Normalise by maximum amplitude

Divide every I and Q sample by the maximum absolute amplitude of that recording. This removes session-level power variation — different recording conditions might capture slightly different signal strengths, and we don't want the model to learn that.

> **Why not subtract the mean?** Subtracting the mean would remove the DC offset, which is itself a potential hardware fingerprint. We deliberately preserve it.

### Step 3 — Segment into 128-sample windows

Split the continuous IQ stream into non-overlapping 128-sample windows. Each window becomes one training example.

128 samples was chosen to balance two things:
- **Enough signal** for the CNN to detect hardware fingerprint patterns
- **Small enough** to keep inference latency low (PR-1 requirement)

At 1 MHz sample rate, 128 samples = 0.128 milliseconds per window.

### Step 4 — Label windows

- DEV01 windows → label 0
- DEV02 windows → label 1

### Step 5 — Inject AWGN noise offline

For each clean recording, create three augmented versions by adding Gaussian noise:

| Tag | SNR level | Purpose |
|---|---|---|
| `clean` | No noise added | High-SNR baseline |
| `SNR20` | 20 dB | Moderate noise |
| `SNR10` | 10 dB | Realistic indoor noise |
| `SNR0` | 0 dB | Challenging noise condition |

> **Why 30 dB is excluded:** Ambient noise in the recording environment contaminated the 30 dB recordings. Those files are unreliable so we don't use them.

---

## Output Structure

```
Preprocessed/
├── S1/
│   ├── BPSK_clean_X.npy    (100,000 windows × 128 samples × 2 channels)
│   ├── BPSK_clean_y.npy    (100,000 labels)
│   ├── BPSK_SNR20_X.npy
│   ├── BPSK_SNR20_y.npy
│   ├── BPSK_SNR10_X.npy
│   ├── BPSK_SNR10_y.npy
│   ├── BPSK_SNR0_X.npy
│   ├── BPSK_SNR0_y.npy
│   └── ... (same for QPSK, GFSK, OOK)
└── S2/
    └── ... (identical structure)
```

**Total:** 4 modulations × 4 SNR levels × 2 files (X + y) × 2 sessions = **64 files**

Each `_X.npy` file: shape `(100,000, 128, 2)`, dtype `float32` — about 97.7 MB  
Each `_y.npy` file: shape `(100,000,)`, dtype `float32` — about 391 KB

---

## Key Design Decisions

| Decision | Reason |
|---|---|
| No mean subtraction | Preserves DC offset as a potential hardware fingerprint |
| Max-amplitude normalisation only | Removes session power variation without destroying fingerprint structure |
| 128-sample windows | Balances fingerprint content vs inference latency |
| Non-overlapping windows | Prevents data leakage between train and test splits |
| Offline AWGN injection | Faster than online augmentation during training; reproducible |
| 30 dB excluded | Ambient noise contamination in those recordings |

---

## How to Use the Preprocessed Data

```python
import numpy as np

# Load BPSK clean data from Session 1
X = np.load('Preprocessed/S1/BPSK_clean_X.npy')   # shape: (100000, 128, 2)
y = np.load('Preprocessed/S1/BPSK_clean_y.npy')   # shape: (100000,)

# X[i] is one 128-sample IQ window
# X[i, :, 0] = I channel
# X[i, :, 1] = Q channel
# y[i] = 0 for DEV01, 1 for DEV02
```
