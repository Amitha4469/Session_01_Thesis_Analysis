# Receiver Hardware Comparison: USRP B200 vs RTL-SDR V3

**QPSK · 900 MHz · 1 Msps — Same Transmitters, Different Receivers**

## 1. Abstract
This sub-repository contains the replication code, analysis, and results for a side-by-side hardware comparison between the USRP B200 and the RTL-SDR V3. The experiment evaluates the impact of different receiver architectures (Superheterodyne vs. Direct-conversion) on capturing QPSK signals, providing baseline hardware signatures for RF fingerprinting models.

## 2. Experiment Setup
We transmitted QPSK signals using two USRP B200s and captured them simultaneously using a USRP B200 and an RTL-SDR V3 to isolate receiver-induced hardware impairments.

| Parameter | USRP Setup | RTL-SDR Setup |
|-----------|-----------|---------------|
| **Receiver** | USRP B200 (serial 3288FAD) | RTL-SDR V3 dongle |
| **Transmitter DEV01** | USRP B200 (serial 3288FF2) | USRP B200 (serial 3288FF2) |
| **Transmitter DEV02** | USRP B200 (serial 3467EEC) | USRP B200 (serial 3467EEC) |
| **Centre frequency** | 900 MHz | 900 MHz |
| **Sample rate** | 1 Msps | 1 Msps |
| **TX gain** | 30 dB | 30 dB |
| **RX gain** | 30 dB (UHD) | RF=30 dB, IF=20 dB, BB=20 dB |
| **AGC** | Disabled | Disabled (gain_mode=False) |
| **DC offset mode** | Disabled | 0 (off) |
| **IQ balance mode** | Disabled | 0 (off) |
| **ADC resolution** | 12-bit | 8-bit |
| **Architecture** | Superheterodyne | Direct-conversion (zero-IF) |
| **Format** | fc32 (complex64) | fc32 (complex64) |

*GRC source files used for capture: `DEV01_QPSK_S3.grc` (USRP) and `DEV01_QPSK_S3_RTL.grc` (RTL-SDR)*

## 3. Data and Directory Layout
The raw IQ `.dat` files were recorded to Google Drive. The script automatically discovers files following this naming convention:
* **USRP Data:** `DEV01_QPSK_S3_R#.dat`
* **RTL-SDR Data:** `DEV0X_QPSK_S3_R#_RTL.dat`

The analysis outputs are saved into `HW_Comparison/figures/` and summarized in `HW_comparison_summary.csv`.

## 4. Analysis Steps & Key Findings
The notebook processes the complex64 IQ data by dropping the first 100,000 transient samples and normalizing by peak amplitude.

### Raw IQ Waveform Comparison
Plotting the I and Q channels reveals the raw signal shapes from both the 12-bit USRP and the 8-bit RTL-SDR.

![Raw IQ Waveforms](figures/raw_iq_waveform.png) 
*(Note: Upload your waveform image to the figures folder and name it raw_iq_waveform.png)*

### Carrier Frequency Offset (CFO) and RMS
Extracted statistical tests show distinct differences in offset and error magnitude between the two receivers:
* **USRP B200 RX:** * CFO mean: **+0.2238 mrad/s**
  * RMS mean: **0.1232**
* **RTL-SDR V3 RX:** * CFO mean: **-0.1983 mrad/s**
  * RMS mean: **0.2871**

### IQ Imbalance Analysis
Because the RTL-SDR V3 uses a direct-conversion (zero-IF) architecture, it inherently introduces IQ imbalance (amplitude and phase mismatch between I and Q branches). 

* **USRP B200 RX Imbalance:**
  * Amplitude: mean = +0.0304 dB, std = 1.1007 dB
  * Phase: mean = +0.3656°, std = 6.9040°
* **RTL-SDR V3 RX Imbalance:**
  * Amplitude: mean = -0.0180 dB, std = 0.9704 dB
  * Phase: mean = +0.0602°, std = 6.6440°

![IQ Imbalance](figures/iq_imbalance.png)
*(Note: Upload your imbalance image to the figures folder and name it iq_imbalance.png)*

## 5. Using these results in RF fingerprinting
The features and figures extracted here act as a hardware-comparison baseline. The IQ recordings and derived feature sets can be used to:
1. Stress-test model robustness to different receiver architectures.
2. Analyse whether per-device signatures learned on USRP recordings carry over to RTL recordings.
3. Build datasets that explicitly include receiver diversity as a factor.

## 6. Acknowledgments
**Authors:** Amitha & Tharangi Madushani
**Supervisor:** Prof. Qinghua Wang, Kristianstad University

## 7. License
MIT License
