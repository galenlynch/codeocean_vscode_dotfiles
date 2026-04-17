# SpikeInterface Preprocessing

## How preprocessing chains work

SI preprocessing is lazy — each step returns a new Recording object that wraps the previous one. No data is copied until `.save()` or `.get_traces()` is called. The chain is serializable to JSON for reconstruction.

```python
# This builds a lazy chain, no data processed yet
rec1 = spre.phase_shift(recording, margin_ms=100)
rec2 = spre.highpass_filter(rec1, freq_min=300, margin_ms=5)
rec3 = spre.common_reference(rec2, reference="global", operator="median")

# This materializes to binary
rec_saved = rec3.save(folder="preprocessed")

# This reconstructs the chain from JSON (lazy again)
rec_loaded = si.load("preprocessed.json", base_folder=data_folder)
```

## Filter implementation (`preprocessing/filter.py`)

- Uses `scipy.signal.iirfilter` for coefficient generation
- Default: Butterworth, order 5, SOS form
- Direction: `forward-backward` (zero-phase via `sosfiltfilt`) — effective order is 2x
- `margin_ms` adds padding to avoid edge effects (reflected padding)
- `HIGHPASS_ERROR_THRESHOLD_HZ = 100` — warns if highpass is below 100 Hz

### Highpass filter
```python
spre.highpass_filter(recording, freq_min=300.0, margin_ms=5.0)
# Butterworth order 5, SOS, zero-phase (sosfiltfilt)
```

### Bandpass filter
```python
spre.bandpass_filter(recording, freq_min=300.0, freq_max=6000.0, margin_ms=5.0)
```

## Common reference (`preprocessing/common_reference.py`)

```python
spre.common_reference(recording, reference="global", operator="median")
# Subtracts the global median across all channels per sample
# Options: reference="global"|"single"|"local", operator="median"|"average"
```

## Bad channel detection (`preprocessing/detect_bad_channels.py`)

```python
spre.detect_bad_channels(recording, method="coherence+psd",
    dead_channel_threshold=-0.5, noisy_channel_threshold=1.0,
    outside_channel_threshold=-0.3, n_neighbors=11)
# Returns: (bad_channel_ids, channel_labels)
# Labels: "dead", "noise", "out", or unlabeled (good)
```

## Motion estimation and correction

```python
# Estimation (saves motion object to folder)
motion = spre.compute_motion(recording, preset="dredge_fast", folder=motion_folder)

# Application (returns new lazy recording)
from spikeinterface.sortingcomponents.motion import interpolate_motion
rec_corrected = interpolate_motion(recording, motion=motion)
```

Motion presets available via `spre.get_motion_presets()`. The AIND pipeline uses `"dredge_fast"`.

## Important: Double-preprocessing in AIND pipeline

When the AIND pipeline runs with default settings:
1. Pipeline applies: phase_shift → highpass(300Hz) → bad_channel_removal → CMR → save_binary
2. KS4 then applies: highpass(300Hz) → whitening → CAR (because `skip_kilosort_preprocessing=False`)

The second highpass is a no-op on already-filtered data. The KS4 CAR is on top of the pipeline's CMR. The whitening is unique to KS4 and is needed for its template matching.
