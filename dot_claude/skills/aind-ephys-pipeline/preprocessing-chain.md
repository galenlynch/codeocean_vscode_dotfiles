# AIND Pipeline: Preprocessing Chain

Source: [aind-ephys-preprocessing](https://github.com/AllenNeuralDynamics/aind-ephys-preprocessing)

## Processing order (exact)

1. **Phase shift** (`spre.phase_shift`, `margin_ms=100`)
   - Only if recording has `inter_sample_shift` property (Neuropixels ADC timing correction)

2. **Temporal filter** (default: highpass)
   - Highpass: `spre.highpass_filter(freq_min=300.0, margin_ms=5.0)` — Butterworth order 5, zero-phase
   - Bandpass option: `freq_min=300.0, freq_max=6000.0, margin_ms=5.0`

3. **Bad channel detection** (`spre.detect_bad_channels`)
   - `method="coherence+psd"`
   - `dead_channel_threshold=-0.5`
   - `noisy_channel_threshold=1.0`
   - `outside_channel_threshold=-0.3`
   - `n_neighbors=11, seed=0`
   - Labels channels as: dead, noise, out (outside brain)
   - If bad channels >= 50% (`max_bad_channel_fraction=0.5`), **entire recording is skipped**

4. **Remove outside-brain channels** (`recording.remove_channels(out_channel_ids)`)

5. **Denoising** (two strategies):
   - **CMR (default)**: `spre.common_reference(reference="global", operator="median")`
   - **Destripe**: interpolate bad channels → highpass spatial filter (`n_channel_pad=60, apply_agc=True, highpass_butter_order=3, highpass_butter_wn=0.01`)
   - If destripe fails (short probes), falls back to CMR

6. **Remove bad channels** (dead + noise) after denoising

7. **Save to binary** (`recording.save()`)
   - Materializes the lazy preprocessing chain to disk
   - This binary is what KS4 receives

8. **Motion estimation** (computed, NOT applied by default)
   - `spre.compute_motion(preset="dredge_fast")`
   - `win_step_um` and `win_scale_um` are normalized to probe span (10% each)
   - `bin_s=1` (default; can be overridden by pipeline parameter)
   - Saved to `motion_{recording_name}/` for visualization
   - `raise_error=False` — motion failure is non-fatal

## Skip conditions

- Recording duration < 120s (`min_preprocessing_duration`): skipped entirely
- Bad channels >= 50%: skipped entirely
- Missing `inter_sample_shift`: phase shift step skipped (not an error)

## What KS4 receives

A binary file containing: phase-shifted → highpass-filtered (300 Hz) → outside-channels-removed → CMR'd → bad-channels-removed data. This is already heavily preprocessed.

KS4 then applies its own: highpass (300 Hz, no-op on already-filtered data) → whitening → CAR.
