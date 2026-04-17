# SpikeInterface Postprocessing & Quality Metrics

## SortingAnalyzer

The central object for postprocessing. Created from a Sorting + Recording pair:

```python
analyzer = si.create_sorting_analyzer(
    sorting=sorting, recording=recording,
    sparse=True, method="radius", radius_um=100,
    return_scaled=True
)
```

- `sparse=True` with `radius_um=100` means each unit only stores waveforms from channels within 100um
- `return_scaled=True` means traces are in uV (not raw int16 ADC values)
- Extensions are computed lazily and cached

## Extensions (computed in dependency order)

```python
# Typical computation order:
analyzer.compute("random_spikes", max_spikes_per_unit=500, method="uniform")
analyzer.compute("noise_levels")
analyzer.compute("waveforms", ms_before=3.0, ms_after=4.0)
analyzer.compute("templates")  # mean of extracted waveforms
analyzer.compute("spike_amplitudes", peak_sign="neg")
analyzer.compute("unit_locations", method="monopolar_triangulation")
analyzer.compute("spike_locations", method="grid_convolution")
analyzer.compute("correlograms", window_ms=50.0, bin_ms=1.0)
analyzer.compute("isi_histograms", window_ms=100.0, bin_ms=5.0)
analyzer.compute("template_similarity", method="l1")  # or "cosine_similarity"
analyzer.compute("template_metrics", upsampling_factor=10, include_multi_channel_metrics=True)
analyzer.compute("principal_components", n_components=5, mode="by_channel_local", whiten=True)
```

### Template computation

- Templates are the **mean** of extracted waveforms (default `operator="average"`)
- `random_spikes` extension controls how many spikes are sampled: `max_spikes_per_unit=500`
- With 500 spikes, template estimation is decent but may miss amplitude modes
- Templates are stored per-unit with sparsity (only nearby channels)

### Template similarity

- `method="l1"` (AIND postprocessing capsule default) or `"cosine_similarity"` (AIND pipeline default)
- **Discrepancy**: the capsule's `params.json` says `l1`, the pipeline's `default_params.json` says `cosine_similarity`. When run via the pipeline (which passes params via `--params`), cosine_similarity is used.
- Used by `remove_redundant_units()` for de-duplication

## Quality metrics

Computed via:
```python
analyzer.compute("quality_metrics", metric_names=[...], qm_params={...})
```

### Key metrics and their parameters (AIND defaults)

| Metric | Key params | What it measures |
|--------|-----------|-----------------|
| `num_spikes` | — | Total spike count |
| `firing_rate` | — | Spikes / duration |
| `presence_ratio` | `bin_duration_s=60` | Fraction of time bins with spikes |
| `snr` | `peak_sign="neg"` | Template peak amplitude / noise level |
| `isi_violation` | `isi_threshold_ms=1.5, min_isi_ms=0` | Hill et al. ISI violation ratio |
| `rp_violation` | `refractory_period_ms=1.0, censored_period_ms=0.0` | Refractory period violations |
| `sliding_rp_violation` | `bin_size_ms=0.25, max_ref_period_ms=10` | Adaptive refractory period estimation |
| `amplitude_cutoff` | `num_histogram_bins=100` | Estimated fraction of missing spikes |
| `amplitude_median` | `peak_sign="neg"` | Median spike amplitude |
| `amplitude_cv` | `percentiles=[5,95], min_num_bins=10` | Amplitude coefficient of variation |
| `synchrony` | `synchrony_sizes=[2,4,8]` | Fraction of spikes in synchronous events |
| `firing_range` | `bin_size_s=5, percentiles=[5,95]` | Range of firing rates across time |
| `drift` | — | Spatial drift of unit position over time |
| `isolation_distance` | — | Mahalanobis distance to nearest cluster |
| `l_ratio` | — | Contamination from nearest cluster |
| `d_prime` | — | Linear discriminability |
| `nearest_neighbor` | `max_spikes=10000, n_neighbors=4` | NN hit/miss rate |
| `silhouette` | `method=["simplified"]` | Cluster separation score |

### Important notes on metrics

- `isi_violation` (Hill method): uses `isi_threshold_ms=1.5` — violations are spikes within 1.5ms. The RATIO is violations/total. The AIND curation threshold is `< 0.5` (50% violations allowed — very permissive).
- `rp_violation`: uses `refractory_period_ms=1.0` — shorter than typical 2ms refractory period
- `amplitude_cutoff`: estimates fraction of the amplitude distribution below detection threshold. `< 0.1` means < 10% estimated missing. This is the STRICTEST criterion in the AIND curation query.
- `presence_ratio`: `> 0.8` means unit must fire in at least 80% of 60s bins
- `sliding_rp_violation` with `bin_size_ms=0.25` is the most sophisticated refractory metric but is NOT used in the default curation query
