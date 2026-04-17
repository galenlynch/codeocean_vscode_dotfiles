# AIND Pipeline: Postprocessing Configuration

Source: [aind-ephys-postprocessing](https://github.com/AllenNeuralDynamics/aind-ephys-postprocessing)

## Processing steps

1. **Create SortingAnalyzer** (sparse, `radius_um=100`, `return_scaled=True`)

2. **Compute initial templates** (for de-duplication)
   - `random_spikes`: max_spikes_per_unit=500, method="uniform"
   - `templates`: default (mean of waveforms)

3. **De-duplication**: `sc.remove_redundant_units(duplicate_threshold=0.9)`
   - Finds unit pairs sharing >90% of spikes (within 0.4ms window)
   - Keeps unit with minimum peak shift (default strategy)
   - Very conservative — only near-identical units removed

4. **Re-create SortingAnalyzer** on de-duplicated units in binary_folder format

5. **Compute all extensions**:
   - `random_spikes`: max_spikes_per_unit=500
   - `noise_levels`: num_chunks=20, chunk_size=10000
   - `waveforms`: ms_before=3.0, ms_after=4.0
   - `templates`: mean waveforms
   - `spike_amplitudes`: peak_sign="neg"
   - `template_similarity`: method depends on invocation (see note below)
   - `correlograms`: window_ms=50.0, bin_ms=1.0
   - `isi_histograms`: window_ms=100.0, bin_ms=5.0
   - `unit_locations`: method="monopolar_triangulation"
   - `spike_locations`: method="grid_convolution"
   - `template_metrics`: upsampling_factor=10, multi_channel=True
   - `principal_components`: n_components=5, by_channel_local, whiten=True

6. **Quality metrics** (18 metrics):
   - num_spikes, firing_rate, presence_ratio, snr, isi_violation, rp_violation, sliding_rp_violation, amplitude_cutoff, amplitude_median, amplitude_cv, synchrony, firing_range, drift, isolation_distance, l_ratio, d_prime, nearest_neighbor, silhouette

7. **Save as zarr** (`postprocessed_{recording_name}.zarr`)

## Template similarity method discrepancy

- Capsule's `params.json`: `"l1"`
- Pipeline's `default_params.json`: `"cosine_similarity"`
- When run via pipeline (params passed via `--params`), cosine_similarity is used
- When capsule runs standalone, l1 is used
- This affects de-duplication behavior (different similarity measure)

## Motion correction at postprocessing (optional)

If `use_motion_corrected=True` (default **False**):
- Loads motion info from preprocessing
- Applies `interpolate_motion()` to recording BEFORE postprocessing
- Templates and spike locations would then be in motion-corrected space
- Currently NOT enabled in default pipeline

## Quality metric parameters (notable)

| Metric | Key Param | Value | Implication |
|--------|-----------|-------|------------|
| `isi_violation` | `isi_threshold_ms` | 1.5 | Standard Hill method |
| `rp_violation` | `refractory_period_ms` | **1.0** | Shorter than typical 2ms |
| `sliding_rp_violation` | `bin_size_ms` | **0.25** | 0.25ms bins — high resolution |
| `amplitude_cutoff` | `num_histogram_bins` | 100 | |
| `presence_ratio` | `bin_duration_s` | 60 | 1-minute bins |
| `synchrony` | `synchrony_sizes` | [2,4,8] | |
