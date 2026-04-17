# KS4 Parameter Reference

Source: `kilosort/parameters.py`. Parameters split into MAIN_PARAMETERS and EXTRA_PARAMETERS.

## Detection parameters

| Parameter | Default | Effect |
|-----------|---------|--------|
| `Th_universal` | 9 | Universal template detection threshold. Higher = fewer spikes, more oversplitting risk. Squared internally (`Cf > Th²`). |
| `Th_learned` | 8 | Learned template detection threshold. Higher = missed spikes. |
| `Th_single_ch` | 6 | Single-channel threshold for learning templates from data |
| `min_template_size` | 10 | Smallest template width (Gaussian sigma in um) |
| `template_sizes` | 5 | Number of multi-scale template sizes |
| `templates_from_data` | True | Learn templates from high-SNR snippets via PCA + KMeans |
| `n_templates` | 6 | Number of data-derived template clusters |
| `n_pcs` | 6 | Number of PC features per spike |
| `nearest_chans` | 10 | Local channel neighborhood for detection |
| `nearest_templates` | 100 | Max template candidates per spike |
| `max_channel_distance` | null | If set, filters out distant template matches |
| `nt` | 61 | Template time samples (~2ms at 30kHz) |
| `nt0min` | null | Auto: `int(20 * nt / 61)` — template peak alignment |

## Drift correction parameters

| Parameter | Default | Effect |
|-----------|---------|--------|
| `nblocks` | 1 | 0 = no drift correction. >1 = multi-block (AIND uses 5) |
| `binning_depth` | 5 | Depth bin size in um for drift histogram |
| `drift_smoothing` | [0.5, 0.5, 0.5] | Gaussian smoothing of drift field |
| `sig_interp` | 20 | Interpolation sigma for drift correction |
| `batch_size` | 60000 | Samples per batch (~2s at 30kHz) |
| `nskip` | 25 | Batches to skip during drift estimation |

## Clustering parameters

| Parameter | Default | Effect |
|-----------|---------|--------|
| `acg_threshold` | 0.2 | Refractory violation fraction. < this = "good" unit. Higher = more permissive. |
| `ccg_threshold` | 0.25 | Cross-CCG threshold for merge candidates |
| `cluster_downsampling` | 20 | Spikes per clustering node. Higher = faster but less precise. |
| `max_cluster_subset` | 25000 | Cap on clustering subset size |
| `cluster_neighbors` | 10 | Graph neighborhood size for nearest-neighbor search |
| `cluster_init_seed` | 5 | Random seed — changes this changes cluster IDs |
| `x_centers` | null | Explicit x-center positions for clustering grid |
| `dmin` | auto | Vertical template spacing. Auto-detected from probe geometry. |
| `dminx` | 32 | Horizontal template spacing (32um for Neuropixels) |

## Preprocessing parameters

| Parameter | Default | Effect |
|-----------|---------|--------|
| `highpass_cutoff` | 300 | Hz. Redundant when pipeline already filters. |
| `whitening_range` | 32 | Channels for whitening matrix computation |
| `artifact_threshold` | null | If set, zeroes out batches with max amplitude above threshold |
| `do_CAR` | True | Common average reference. Set via SI wrapper. |

## Post-processing parameters

| Parameter | Default | Effect |
|-----------|---------|--------|
| `duplicate_spike_ms` | 0.25 | Same-cluster spike dedup window in ms |
| `max_peels` | 100 | Matching pursuit iterations per batch (EXTRA_PARAMETERS) |

## SI wrapper parameters (not KS4 native)

| Parameter | Default | Effect |
|-----------|---------|--------|
| `skip_kilosort_preprocessing` | False | Skip KS4's internal highpass + whitening |
| `do_correction` | True | Enable/disable drift correction |
| `keep_good_only` | False | Only return "good" units |
| `use_binary_file` | True | Force binary file for KS4 input |
| `delete_recording_dat` | True | Clean up temp binary after sorting |
