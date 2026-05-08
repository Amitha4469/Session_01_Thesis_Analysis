# Signal Analysis — Session 1 & Session 2

**Notebooks:**
- `rf_fingerprint_full_analysis.ipynb` — Session 1 (14 sections)
- `rf_fingerprint_s2_v2_analysis.ipynb` — Session 2 + cross-session stability

This is the foundation of the whole thesis. Before training any model, we first measured the physical hardware fingerprints directly to understand what the CNN would actually be learning.

---

## What We Measured

Every radio transmitter has manufacturing imperfections that create unique distortions in its signal. We measured five types:

| Impairment | What it is | How stable |
|---|---|---|
| Carrier Frequency Offset (CFO) | Oscillator inaccuracy — device transmits at slightly wrong frequency | Very stable |
| IQ Imbalance | Amplitude/phase mismatch in the modulator | Stable |
| Phase Noise | Oscillator instability | Stable |
| RMS Amplitude | Hardware power level differences | Stable |
| DC Offset | Direct current bias in the signal | Stable but tiny |

---

## Session 1 Analysis — Key Findings

### CFO (Carrier Frequency Offset)

| Modulation | DEV01–DEV02 separation | σ within device | Reliable? |
|---|---|---|---|
| BPSK | 887 Hz | up to 634 Hz | ✅ Yes |
| OOK | 197 Hz | < 110 Hz | ✅ Yes |
| QPSK | ~45,000 Hz | ~45,000 Hz | ❌ No — symbol phase jumps dominate |
| GFSK | ~13,000 Hz | ~13,000 Hz | ❌ No — continuous phase transitions |

For BPSK and OOK, CFO is the dominant fingerprint. For QPSK and GFSK, the CFO estimator fails because symbol transitions overwhelm the phase rotation signal. The CNN needs to learn RMS amplitude for those instead.

### RMS Amplitude

| Modulation | Fisher Discriminant Ratio | Reliable? |
|---|---|---|
| QPSK | 22.8 | ✅ Strong |
| GFSK | 15.1 | ✅ Strong |

DEV02 consistently shows higher RMS than DEV01 for QPSK and GFSK. Stable and directional across both sessions.

### DC Offset

Ruled out as a fingerprint — both devices produced nearly identical DC offsets (difference of 0.000001). Not discriminative.

---

## Session 2 Analysis — Cross-Session Stability (ER-4)

Session 2 was recorded on a different day with the hardware physically repositioned. The key test is whether the fingerprints remain stable across sessions — if they don't, the classifier is memorising session-specific noise rather than hardware impairments.

**Result: PASS.** CFO separation and RMS amplitude differences were consistent between S1 and S2. The fingerprints are stable across days.

This directly satisfies ER-4 (Train/test separation across separate sessions) from the system requirements.

---

## What This Means for the CNN

The modulation-dependent feature dominance is not a problem for the CNN — the model receives raw IQ and learns whichever feature separates the two devices automatically:

- BPSK/OOK → CNN learns CFO-based patterns
- QPSK/GFSK → CNN learns RMS amplitude patterns

Both point in a consistent direction. The CNN doesn't need to know which feature to use — it figures it out.

---

## Notebook Structure

**Session 1 notebook (14 sections):**
1. Data inventory and health check
2. IQ sample visualisation
3. Power spectral density
4. CFO estimation
5. RMS amplitude analysis
6. DC offset analysis
7. IQ imbalance analysis
8. Feature separability (Fisher discriminant ratio)
9. Cross-modulation stability
10. Signal quality metrics
11. Per-device fingerprint summary
12. Heatmaps and visualisations
13. Cross-session preview
14. GitHub export

**Session 2 notebook (15 sections):**
Same structure as S1, plus Section 15: cross-session fingerprint stability test comparing S1 and S2 measurements directly.
