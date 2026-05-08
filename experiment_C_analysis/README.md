# Experiment C — One-Class Authenticator

**Notebooks:**
- `experiment_C2_oneclass_embedding_svm.ipynb` — standard embedding + OC-SVM
- `experiment_C3_DANN_oneclass.ipynb` — DANN modulation-invariant embedding + OC-SVM

---

## The Problem We're Solving

Experiments A and B are supervised classifiers — they need examples of both DEV01 and DEV02 during training. In a real deployment, you might only have the legitimate device. You enrol it, then reject anything that doesn't match its hardware fingerprint — without ever having seen the impostor's signal.

That's one-class authentication.

---

## The Approach

```
IQ window (128×2)
      ↓
Frozen 1D CNN (from Experiment A)
      ↓
64-dim embedding — hardware fingerprint space
      ↓
One-Class SVM (trained on DEV01 only)
      ↓
ACCEPT / REJECT
```

The CNN backbone is unchanged from Experiment A. The OC-SVM draws a boundary around DEV01's cluster. Anything outside gets rejected.

---

## C2 — Standard Embedding

Uses the Experiment A binary classifier embedding directly.

### TAR (True Accept Rate — DEV01 accepted)

| Modulation | Clean | SNR 20 dB | SNR 10 dB | SNR 0 dB |
|---|---|---|---|---|
| BPSK | 99.60% | 99.51% | 49.17% | 0.00% |
| QPSK | 100.00% | 100.00% | 31.66% | 0.00% |
| GFSK | 99.90% | 99.73% | 0.00% | 0.00% |
| OOK | 93.76% | 92.16% | 60.65% | 0.01% |

### FAR (False Accept Rate — DEV02 accepted, target ≤ 5%)

| Modulation | Clean | SNR 20 dB | SNR 10 dB | SNR 0 dB |
|---|---|---|---|---|
| BPSK | 34.50% ❌ | 30.53% ❌ | 2.63% ✅ | 0.00% ✅ |
| QPSK | 25.20% ❌ | 22.84% ❌ | 4.01% ✅ | 0.00% ✅ |
| GFSK | 27.47% ❌ | 26.19% ❌ | 0.00% ✅ | 0.00% ✅ |
| OOK | 72.03% ❌ | 67.63% ❌ | 24.45% ❌ | 0.00% ✅ |

**AUC: 0.6115**

![C2 TAR/FAR heatmap](https://drive.google.com/uc?export=view&id=1huXXSFLqNbRGW4drEomJW6_8jLx8hhxE)

![C2 ROC curve](https://drive.google.com/uc?export=view&id=193SdFlgmUfC9SjhQWoyv2L8m2Ly___tm)

![C2 embedding space t-SNE](https://drive.google.com/uc?export=view&id=1m3Y28hJcq2pHQ12FL2KW13G_lMoiqjbq)

### Why it fails at clean/SNR20

The Experiment A model was trained on BPSK only. Its embedding works for BPSK but the other three modulations land in different regions. A single OC-SVM boundary can't cleanly separate DEV01 from DEV02 across four different modulation regions simultaneously.

---

## C3 — DANN with Gradient Reversal

The fix: force the embedding to be modulation-invariant using a Domain-Adversarial Neural Network (DANN). A Gradient Reversal Layer (GRL) makes the backbone learn to make modulation type unpredictable from the embedding, while keeping device identity predictable.

```
IQ (128×2)
    ↓
1D CNN backbone
    ↓
Dense-64 embedding ← adversarially forced to ignore modulation
   /              \
Device head        Modulation head + GRL
DEV01/DEV02        BPSK/QPSK/GFSK/OOK ← backbone learns to fool this
```

At inference, the modulation head is discarded. Same lightweight backbone, same inference path.

### Training behaviour

![DANN training curves — device accuracy up, modulation accuracy down](https://drive.google.com/uc?export=view&id=1hGz31Ump7BDDqHnuglhe9Iz2KfRmM4fD)

| Signal | Value | Target |
|---|---|---|
| Device head accuracy | 78.75% | > 85% |
| Modulation head accuracy | 52.06% | ≈ 25% (random) |

Modulation accuracy dropped from ~95% to 52% — the GRL partially stripped modulation information. Not all the way to 25%, but significant progress.

### C3 Results

![C3 TAR/FAR heatmap](https://drive.google.com/uc?export=view&id=1E-8GrIwG5YIWVk1R8i9dipX1DiOJOed9)

![C3 ROC curve](https://drive.google.com/uc?export=view&id=1KR4w6lBEfUoUBW3qp2IHA2vb4f25x9ax)

**Global OC-SVM — AUC: 0.7484** (up from 0.6115 in C2)

| Modulation | TAR Clean | FAR Clean | SR-2 ≤ 5% |
|---|---|---|---|
| BPSK | 41.02% | 0.01% | ✅ |
| QPSK | 41.71% | 19.07% | ❌ |
| GFSK | 66.86% | 21.66% | ❌ |
| OOK | 78.57% | 78.76% | ❌ |

**Per-Modulation OC-SVM**

| Modulation | TAR Clean | FAR Clean | SR-2 ≤ 5% |
|---|---|---|---|
| BPSK | 76.87% | 0.02% | ✅ |
| QPSK | 57.38% | 14.31% | ❌ |
| GFSK | 4.36% | 1.78% | ✅ |
| OOK | 2.60% | 5.69% | ❌ |

---

## The Full Progression

| Approach | AUC | BPSK FAR | Cross-modulation |
|---|---|---|---|
| C2 — Standard embedding | 0.6115 | 34.50% | ❌ |
| C3 — DANN global SVM | 0.7484 | 0.01% | ⚠️ Partial |
| C3 — DANN per-mod SVM | — | 0.02% | BPSK ✅, GFSK ✅ (low TAR) |

---

## Why Full Cross-Modulation One-Class Fails

The 43,874-parameter architecture can't simultaneously keep device fingerprints discriminable and make modulation type completely unpredictable. The GRL got us from ~95% to 52% mod accuracy — real progress — but not the 25% needed for a global SVM to work cleanly.

OOK has an additional irreducible problem: ~50% of windows are silent (transmitter OFF). No signal means no hardware fingerprint. No algorithm can authenticate a device from a zero-amplitude window.

Full modulation-agnostic one-class authentication requires either contrastive/triplet training or a deeper architecture. Left as future work.

---

## Supervised vs One-Class

| Approach | Accuracy / AUC | FAR | Cross-modulation |
|---|---|---|---|
| Supervised binary (Exp A + B) | 97–100% | 0.00% | ✅ Full |
| One-class DANN (Exp C3) | AUC 0.75 | 0.02% (BPSK) | ⚠️ Partial |

When you have impostor training data, the supervised approach is significantly better. One-class authentication is the harder problem — these results show where a lightweight architecture hits its limits.
