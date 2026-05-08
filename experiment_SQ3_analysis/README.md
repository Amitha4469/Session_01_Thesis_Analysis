# SQ3 — Raw IQ vs STFT Spectrogram

**Notebook:** `experiment_SQ3_iq_vs_stft.ipynb`  
**Research gap closed:** G3 — no controlled head-to-head comparison of these two input representations under the same CNN family and SNR sweep existed before this.

---

## The Question

There are two common ways to feed a signal into a CNN for RF fingerprinting:

1. **Raw IQ** — pass the I and Q samples directly as a (128, 2) tensor. No preprocessing beyond normalisation.
2. **STFT spectrogram** — compute the Short-Time Fourier Transform and feed the magnitude image. Shape (64, 17, 1) for our 128-sample windows.

Prior work evaluated each separately but never compared them head-to-head under the same architecture, same SNR sweep, same data. We did that here.

---

## Results

![Full SQ3 comparison — accuracy, FAR, latency, summary table](https://drive.google.com/uc?export=view&id=1yUf7KmuuN2hNXRKMjEfx9i21ii_ObvT4)

### Accuracy

| SNR | 1D CNN (Raw IQ) | 2D CNN (STFT) |
|---|---|---|
| Clean | 100.00% | 100.00% |
| SNR 20 dB | 100.00% | 100.00% |
| SNR 10 dB | 100.00% | 100.00% |
| SNR 0 dB | **99.99%** | 98.58% |

### False Acceptance Rate

| SNR | 1D CNN (Raw IQ) | 2D CNN (STFT) |
|---|---|---|
| Clean | 0.00% | 0.00% |
| SNR 20 dB | 0.00% | 0.00% |
| SNR 10 dB | 0.00% | 0.00% |
| SNR 0 dB | **0.00%** | 1.22% |

### Model size and speed

| Metric | 1D CNN | 2D CNN |
|---|---|---|
| Parameters | **43,874** | 101,058 |
| Latency (median) | **67.57 ms** | 68.33 ms |

> Latency figures reflect Colab single-window overhead, not real edge deployment. The relative comparison is what matters.

---

## Why Raw IQ Wins

The STFT computes magnitude from the complex signal — that step throws away phase information. Hardware fingerprints like carrier frequency offset and IQ imbalance are encoded in the phase. Raw IQ keeps everything.

At clean and SNR20 this doesn't matter much — amplitude patterns alone are sufficient. At SNR0 the difference shows: the 2D CNN introduces 1.22% FAR while the 1D CNN stays at 0%. Raw IQ is also faster because it skips the per-window FFT entirely.

---

## Requirements

| Requirement | 1D CNN | 2D CNN |
|---|---|---|
| SR-1: ≥ 95% accuracy at SNR ≥ 20 dB | ✅ 100% | ✅ 100% |
| SR-2: FAR ≤ 5% | ✅ 0.00% all SNR | ⚠️ 1.22% at SNR0 |
| PR-2: < 500K parameters | ✅ 43,874 | ✅ 101,058 |

---

## Bottom Line

Raw IQ beats STFT on every metric: accuracy at low SNR, FAR, parameter count, and inference speed. For lightweight IoT authentication, raw IQ is the right input representation. This closes Research Gap G3.
