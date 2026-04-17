# SpikeInterface / KS4 Data Conventions

Accumulated from hands-on work with AIND Neuropixels 2.0 data. These are empirical observations that supplement the codebase-level documentation.

## KS4 phy file conventions

- `amplitudes.npy` = **negative peak voltage on peak channel** per spike (uV), NOT a template scaling factor
- `templates.npy` (n_units, n_samples, n_sparse_channels) — sparse. Use `template_ind.npy` (n_units, n_sparse_channels) to map to full channel indices. Padded with -1 for units with fewer sparse channels.
- `spike_times.npy` = sample indices relative to the `.dat` file (starts at 0), NOT absolute NWB timestamps
- `spike_clusters.npy` = cluster assignment per spike
- To get a template scaling factor: `scale = spike_amp / template_trough_value`

## Template polarity gotcha

- KS4 can create templates that are **positive-going** on the peak channel (dendritic return currents)
- SI's mean waveform faithfully reproduces this
- Spike-triggered averaging from the `.dat` gives correct polarity because it averages actual traces
- UnitRefine (RandomForest on scalar metrics, NOT a CNN) catches ~68% of dendritic units as MUA but misses ~32%
- A positive-going template with negative amplitudes is NOT a sign convention bug — it's a real pathology (dendritic split)

## SpikeInterface amplitude/template conventions

- `spike_amplitudes` extension with `peak_sign='neg'` = negative peak of each spike on peak channel (same as KS4's amplitudes.npy)
- SI zarr templates with `return_in_uV=True` = mean waveform in uV; gain is `gain_to_uV` from `rec_attributes` (2.34 for NP2.0)
- `template_similarity` extension = L1-based (capsule default) or cosine (pipeline default), `support='union'`, range [0, 1]

## Sample space / time alignment

- Motion `temporal_bins_s` are in absolute NWB time (~14.4M seconds from epoch); subtract `temporal_bins_s[0][0]` to align with recording t=0
- Phy `.dat` starts at sample 0; sorting spike times are in this space
- The S3 zarr has absolute timestamps; the `.dat` drops them
- `recording_corrected.dat` uses the same sample space as the phy `.dat` if written from the same source

## Recording provenance chain

- `SortingAnalyzer.recording` follows a provenance chain that can fail if raw data isn't at the expected path. This is the **most common failure** when reusing sorted assets on a different machine or Code Ocean instance. `analyzer.has_recording()` returns `False`, and accessing `analyzer.recording` raises `ValueError`.
- Use `analyzer.set_temporary_recording(recording)` to override with a reconstructed copy.
- The AIND pipeline saves the full preprocessing chain as JSON at `preprocessed/<recording_name>.json` in the sorted asset. The chain is: `ZarrRecording -> SelectSegment -> PhaseShift -> HighpassFilter -> ChannelSlice -> CommonReference`. The root path is relative (`../data/...`). Rewrite it to point at the current data mount and call `si.load_extractor()` to replay.
- The `pl-oversplitting-analysis-capsule` has `data_discovery.reconstruct_recording()` that automates this.

## Motion correction operational notes

- `interpolate_motion` requires float32 input (`astype(recording, np.float32)`)
- `border_mode='force_extrapolate'` keeps all 384 channels (default removes ~10 edge channels)
- Kriging sigma=20um introduces ~+0.02 amp_cross bias at adjacent channels (negligible for 0.5 threshold)
- `Motion` class: `from spikeinterface.core.motion import Motion` (SI 0.104+)
