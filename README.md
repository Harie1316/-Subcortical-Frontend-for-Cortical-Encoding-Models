# Subcortical Frontend for Cortical EEG Encoding Models

A differentiable, biophysically-grounded subcortical auditory frontend for predicting cortical EEG responses to continuous speech.

> Most cortical EEG encoding models for speech use log-mel spectrograms as input features. This project tests whether interposing a **learned, biophysically-motivated subcortical stage** between the audio waveform and the cortical encoder improves EEG prediction on the Broderick et al. (2019) Cocktail Party dataset (33 subjects, 128-channel EEG).

## Pipeline

```
raw audio (16 kHz)
    │
    ▼
learned subcortical student  ── trained to mimic a biophysical
(1D dilated CNN, 240k params)   gammatone–IHC–ANF cochlear cascade
    │
    ▼
1-channel population FFR (8 kHz)
    │
    ▼
linear TRF cortical encoder
(ridge regression, ±100/500 ms lags)
    │
    ▼
128-channel EEG prediction (128 Hz)
```

The learned subcortical stage is a differentiable surrogate of a gammatone–IHC–ANF cochlear cascade with cochlear-travel-time-corrected across-CF summation and a unitary-response convolution. It produces a single-channel population frequency-following response (FFR) — exactly the signal that subcortical EEG measurements record from the auditory brainstem.

## Headline result

Across N = 33 subjects, evaluated by leave-one-trial-out cross-validation:

| Frontend | Mean Pearson *r* | Std |
| --- | --- | --- |
| Mel spectrogram (28 ch) baseline   | +0.0115 | 0.0119 |
| **Learned subcortical (1 ch)**     | **+0.0243** | **0.0158** |
| Cochlea oracle (64 CFs)            | +0.0093 | 0.0102 |

The learned subcortical frontend predicts EEG **2.1× better** than the mel-spectrogram baseline (paired *t*(32) = +8.15, *p* = 2.6×10⁻⁹; Wilcoxon *p* = 1.0×10⁻⁸; **31 of 33 subjects improved**).

![Main results](figures/fig1_main_results.png)

*Panel A: Three-frontend comparison. The learned subcortical frontend significantly outperforms both the spectrogram baseline and the high-dimensional cochlea oracle. Panel B: Five-way ablation isolating the contributions of dimensionality and biological supervision.*

## Where on the scalp does the improvement live?

![Topography](figures/fig2_topography.png)

The subcortical frontend's advantage over the spectrogram is concentrated over central-parietal channels (panel C), exactly where auditory cortex projects on the scalp. This rules out generic noise reduction and demonstrates the improvement is auditory-cortex-specific.

## Ablation

| Comparison | Δ*r* | *t*(32) | *p* (paired) | *n* improved |
| --- | --- | --- | --- | --- |
| Learned subcortical − Mel spectrogram     | +0.0128 | +8.15  | 2.6×10⁻⁹  | 31/33 |
| Learned subcortical − Random student      | +0.0158 | +10.13 | 1.7×10⁻¹¹ | 32/33 |
| Learned subcortical − Broadband envelope  | +0.0020 | +4.29  | 1.5×10⁻⁴  | 25/33 |
| Random student − Mel spectrogram          | −0.0030 | −1.92  | 0.063     | 13/33 |
| Broadband envelope − Mel spectrogram      | +0.0108 | +6.54  | 2.3×10⁻⁷  | 29/33 |
| Cochlea oracle − Mel spectrogram          | −0.0022 | −2.02  | 0.052     | 12/33 |

The ablation isolates two distinct contributions:

1. **Dimensionality.** Any biologically-relevant 1D signal — even a simple broadband Hilbert envelope — beats the 28-channel mel spectrogram in this regime, because ridge regression on a 28×77 weight matrix is harder to regularise than on a 1×77 matrix.
2. **Biology.** Controlling for dimensionality (versus broadband envelope), the cochlea+brainstem-shaped output of the trained subcortical model is still significantly better (Δ*r* = +0.002, *p* = 1.5×10⁻⁴). A randomly-initialised student with the same architecture but no FFR supervision is **not** better than the spectrogram (Δ*r* = −0.003, *p* = 0.063), confirming the biological supervision target is doing real work.

The high-dimensional cochlea oracle (64 CFs × 77 lags = 4,928 weights per channel) actually performs slightly worse than the spectrogram. This is the bias–variance tradeoff in action: the brain doesn't transmit 30,000 auditory-nerve fibres' worth of signal to cortex; it integrates them into a low-dimensional population response. **That low-dimensional integrated representation — the population FFR — is the right feature for cortical encoding.**

## Subcortical student

![Subcortical pipeline](figures/fig3_subcortical_pipeline.png)

*Panel A: Biophysical cochleagram (auditory nerve firing rate across 64 characteristic frequencies). Panel B: The learned student model (blue) reproduces the biophysical teacher (black) with mean held-out Pearson r = 0.9957 ± 0.0003 on 6 unseen trials.*

The student is a dilated 1D CNN (WaveNet-style residual stack, 240,489 parameters, ~32 ms receptive field at 16 kHz) trained to regress the teacher FFR. Architecture summary in [`MODEL_CARD.md`](MODEL_CARD.md).

| Metric | Value |
| --- | --- |
| Parameters | 239,489 |
| Receptive field | 513 audio samples (32.1 ms) |
| Training epochs | 50 |
| Loss | Time L1 + multi-resolution spectral L1 (80–1500 Hz) |
| Validation Pearson *r* (best epoch) | 0.9951 |
| **Held-out test Pearson *r*** | **0.9957 ± 0.0003** (n = 6 trials) |

## Reproducing

The notebook runs end-to-end on a single Google Colab T4 in ~4 hours.

1. **Open** [`Subcortical_Frontend_for_Cortical_Encoding_Models.ipynb`](Subcortical_Frontend_for_Cortical_Encoding_Models.ipynb) in Colab with the T4 GPU runtime.
2. **Get the data.** The notebook expects the Broderick et al. (2019) Cocktail Party dataset (a single 2.6 GB ZIP). The first cell downloads it via `gdown` — update the file ID to your own Google Drive copy.
3. **Run all cells in order.** Each section is self-contained and produces verification output before the next section runs.

Pipeline phases (each maps to a numbered section in the notebook):

| Section | What it does | Wall time on T4 |
| --- | --- | --- |
| 1. Environment + dataset | Download + extract | ~2 min |
| 2. EEG preprocessing | Re-reference, 1–8 Hz bandpass, z-score | ~5 min |
| 3. Stimulus synthesis | espeak-ng word-aligned TTS | ~1 min |
| 4. Cochlear teacher | Generate FFR targets for all 60 trials | ~15 s |
| 5. Subcortical student | Train 1D CNN to regress teacher FFR | ~47 min |
| 6. Cortical encoding | Three-way TRF comparison | ~12 min |
| 7. Ablation | Add random-student and envelope controls | ~2 min |
| 8. Final figures | Boxplots, topography, pipeline figure | ~3 min |

## Stack

Python 3.11, PyTorch 2.11 (CUDA 12.8), MNE-Python 1.7, NumPy 2.0, SciPy 1.16, librosa 0.10, soundfile, h5py, eSpeak-NG.

## Methods notes

**EEG preprocessing.** Re-reference to the average of the two mastoid channels (the standard reference for this dataset). FIR bandpass 1–8 Hz, zero-phase (cortical TRF band). Per-subject z-score across all of that subject's runs. The source `.mat` files have an axis-order inconsistency in 22 of 984 runs (all in Subject 6), which Cell 6 of the notebook diagnoses and Cell 7 handles robustly.

**Stimulus synthesis.** The original Broderick release does not redistribute the audiobook recordings (likely a copyright issue with the commercial audiobooks). Audio is synthesised with `espeak-ng` formant synthesis: each content word from the dataset's word-level annotations is rendered at a neutral rate, then time-stretched to fit its exact onset–offset window, with silence in between. The resulting acoustics have clear formant structure, harmonics, and consonant bursts, but lower envelope density than natural speech. This means absolute EEG-prediction correlations are lower than published numbers obtained with the original recordings, but the within-study comparisons across frontends are valid and controlled — all frontends see identical audio.

**Biophysical cochlea.** Gammatone filterbank (64 CFs, 80 Hz – 8 kHz, ERB-spaced) → half-wave rectification + power compression with α = 0.3 (IHC transduction) → 1 kHz lowpass via short exponential kernel (IHC membrane filter) → tanh saturation (ANF firing rate). To synthesise the population FFR, per-CF firing rates are delayed by cochlear travel times (5 ms × (1000/CF)^0.7, clamped to [0.5, 12] ms), summed across CFs, convolved with a damped 220 Hz unitary response (Dau, 2003), and downsampled to 8 kHz.

**Subcortical student.** Dilated 1D CNN (WaveNet-style), 8 residual blocks with dilations 1, 2, 4, ..., 128, channel width 64, kernel size 3, gated activation. Final stride-2 conv handles the 16 kHz → 8 kHz rate change. Loss is time-domain L1 + multi-resolution spectral L1 (FFT sizes 256/512/1024, restricted to 80–1500 Hz). AdamW (lr=1e-3, wd=1e-4), cosine schedule, 50 epochs, gradient clip at 1.0. Train on 4-second random crops, 48 trials train / 6 validation / 6 test, stratified by story.

**Cortical TRF.** Closed-form ridge regression on GPU. Lag window −100 ms to +500 ms (77 lags at 128 Hz). Per-subject leave-one-trial-out CV. Inner CV alpha search over [10², 10³, 10⁴] using the next-trial as inner validation. The TRF is implemented in PyTorch on the T4 (rather than CPU numpy) because at 128 channels × 4928 features the Gram matrix and solve are too expensive in float64 numpy.

## Limitations and future work

- **Synthesised stimuli.** TTS audio is acoustically simpler than the original recordings (lower envelope density, no prosody, no speaker variability). Absolute *r* values are correspondingly lower than published Broderick benchmarks (~0.07–0.10), though the relative effects are valid.
- **Linear cortical encoder.** A learned CNN cortical encoder could close more of the gap to the cochlea-oracle ceiling and would be the natural next step.
- **Simplified cochlea.** A gammatone–IHC–ANF cascade lacks active cochlear amplification and some nonlinearities of a full transmission-line model (Verhulst, Altoè & Shera 2018). The CoNNear pre-trained Verhulst surrogate would be a drop-in upgrade if its weights become available again on PyPI.
- **No subcortical EEG.** This pipeline is designed to bridge from raw audio to *cortical* EEG. Validating the learned FFR against actual subcortical EEG recordings (e.g. Bharadwaj or Maddox lab continuous-speech FFR data) is an obvious extension.

## Repository layout

```
.
├── Subcortical_Frontend_for_Cortical_Encoding_Models.ipynb   # full pipeline
├── README.md                                                  # this file
├── MODEL_CARD.md                                              # student model spec
├── figures/
│   ├── fig1_main_results.png                                  # boxplots
│   ├── fig2_topography.png                                    # scalp maps
│   └── fig3_subcortical_pipeline.png                          # cochleagram + FFR
├── results/
│   ├── per_trial_results.csv                                  # every (frontend, subject, trial) → r
│   └── results_summary.json                                   # all reported numbers, machine-readable
└── checkpoints/
    └── subcortical_student_best.pt                            # trained student weights
```

## Reference

Broderick, M. P., Anderson, A. J., Di Liberto, G. M., Crosse, M. J., & Lalor, E. C. (2019). Electrophysiological correlates of semantic dissimilarity reflect the comprehension of natural, narrative speech. *Current Biology*. Dataset on Dryad: [doi.org/10.5061/dryad.070jc](https://doi.org/10.5061/dryad.070jc).
