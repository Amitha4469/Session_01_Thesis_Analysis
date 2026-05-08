# Experiment B — Multi-Modulation Device Fingerprinting

**Notebook:** `experiment_B_multimodulation.ipynb`  
**Model:** `Models/ExperimentB/B1_combined_model.keras`

---

## The Question

Experiment A proved the classifier works for BPSK. But BPSK is one modulation. Real IoT devices use different modulations depending on the protocol and the environment. Can the same lightweight CNN authenticate devices across BPSK, QPSK, GFSK, and OOK — using one trained model?

---

## Two Parts

**B1 — Combined model.** Train on all four modulations at once. See if one model can handle all of them.

**B2 — Leave-One-Modulation-Out (LOMO).** Remove one modulation from training entirely, then test on it. If the fingerprint is truly hardware-based and modulation-agnostic, the classifier should still work. This is the harder test.

Both parts use the same training strategy as Experiment A: multi-session (S1 + S2) + noise augmentation (clean + SNR10 + SNR0).

---

## B1 Results — Combined Model

Evaluated on Session 2, which the model never saw during training.

| Modulation | Accuracy (Clean) | Notes |
|---|---|---|
| BPSK | 100.00% | Perfect |
| QPSK | 97.85% | High despite unreliable CFO estimation |
| GFSK | 98.36% | RMS amplitude dominates as fingerprint |
| OOK | 70.46% | Limited by 50% silent windows |

OOK is the outlier. OOK works by switching the signal ON and OFF — roughly half the windows are silent. During those OFF periods there's literally no signal and therefore no hardware fingerprint. The classifier sees near-identical zeros for both devices. That's not a model failure, it's a signal structure problem.

QPSK and GFSK are interesting. Signal analysis showed CFO estimation is unreliable for both — symbol phase jumps overwhelm the CFO signal. But the model still achieves 97–98% accuracy. It learned to use RMS amplitude differences instead (Fisher discriminant ratios: QPSK=22.8, GFSK=15.1). The CNN adapts to whatever separating feature exists.

![B1 training curves](https://drive.google.com/uc?export=view&id=1z75A1-M5INL-oGu2vipNdf_AComnZ_xF)

![B1 per-modulation accuracy heatmap](https://drive.google.com/uc?export=view&id=1a6My-s196HXnX8F9nKQzuNWtmXqxnSE_)

---

## B2 Results — Leave-One-Modulation-Out

| Held-out modulation | Accuracy |
|---|---|
| BPSK held out | ~50% — random |
| QPSK held out | 38.75% — below random |
| GFSK held out | ~50% — random |
| OOK held out | ~63% |

Every modulation fails when excluded from training. Zero cross-modulation transfer. QPSK actually drops below chance, which means the model confidently predicts the wrong class.

This tells you something concrete: the fingerprint manifold is modulation-specific. BPSK fingerprints and QPSK fingerprints live in different regions of the embedding space, and training on one doesn't help you recognise the other. All four modulations need to be in training.

![B2 LOMO heatmap](https://drive.google.com/uc?export=view&id=1-aCmLzGY00AdVc3tcBOEfzoNMeUr5QhH)

![B1 vs LOMO comparison](https://drive.google.com/uc?export=view&id=1MR2p2SCHK6MCh_RlAPII8z-0UTj5uGt5)

---

## What the Signal Analysis Said

| Modulation | Dominant fingerprint | Why |
|---|---|---|
| BPSK | CFO (887 Hz separation) | Slow symbol transitions let phase rotation accumulate cleanly |
| OOK | CFO (197 Hz separation) | Pulse envelope preserves oscillator offset |
| QPSK | RMS amplitude (FDR = 22.8) | Symbol phase jumps swamp the CFO estimator |
| GFSK | RMS amplitude (FDR = 15.1) | Continuous phase transitions do the same |

---

## Requirements

| Requirement | Result |
|---|---|
| SR-1: Accuracy ≥ 95% at SNR ≥ 20 dB | BPSK 100%, QPSK 97.85%, GFSK 98.36% ✅ |
| SR-2: FAR ≤ 5% | All modulations ✅ |
| PR-2: Model < 500K parameters | 43,874 ✅ |

---

## Bottom Line

One 43,874-parameter model handles BPSK, QPSK, and GFSK at 97–100% accuracy. OOK plateaus at 70% due to signal structure, not the classifier. Cross-modulation transfer doesn't happen on its own — all four modulations must be present in training.
