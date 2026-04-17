# AIND Pipeline: Kilosort4 Configuration

Source: [aind-ephys-spikesort-kilosort4](https://github.com/AllenNeuralDynamics/aind-ephys-spikesort-kilosort4)

## KS4 parameters (as used by AIND)

| Parameter | AIND Value | KS4 Default | Notes |
|-----------|-----------|-------------|-------|
| `batch_size` | 60000 | 60000 | ~2s at 30kHz |
| `nblocks` | **5** | 1 | Multi-block drift correction (AIND-specific) |
| `Th_universal` | 9 | 9 | |
| `Th_learned` | 8 | 8 | |
| `do_CAR` | true | true | On top of pipeline CMR |
| `nt` | 61 | 61 | ~2ms template |
| `nskip` | 25 | 25 | |
| `whitening_range` | 32 | 32 | |
| `highpass_cutoff` | 300 | 300 | Redundant (data already filtered) |
| `binning_depth` | 5 | 5 | |
| `drift_smoothing` | [0.5, 0.5, 0.5] | [0.5, 0.5, 0.5] | |
| `dminx` | 32 | 32 | |
| `templates_from_data` | true | true | |
| `n_templates` | 6 | 6 | |
| `n_pcs` | 6 | 6 | |
| `acg_threshold` | 0.2 | 0.2 | |
| `ccg_threshold` | 0.25 | 0.25 | |
| `cluster_downsampling` | 20 | 20 | |
| `duplicate_spike_ms` | 0.25 | 0.25 | |
| `skip_kilosort_preprocessing` | **false** | false | Double preprocessing |
| `keep_good_only` | **false** | false | Keeps all units |
| `artifact_threshold` | null (= inf) | null | No artifact rejection |

## Key logic in sorting capsule

1. **Multi-segment concatenation**: If recording has multiple segments, they are concatenated before sorting and split back after. This loses segment boundaries within KS4.

2. **Low-channel drift disable**: If `num_channels < MIN_DRIFT_CHANNELS` (default 96 in pipeline, 64 in capsule), drift correction is disabled (`do_correction=False`).

3. **Post-sort cleanup**:
   - `sorting.remove_empty_units()` — removes KS4 units with 0 spikes
   - `sc.remove_excess_spikes(sorting, recording)` — removes spikes beyond recording length (edge case)

4. **Output**: Saved as SI sorting object (zarr or numpy folder format), not phy format.

## What AIND changed from KS4 defaults

The only significant difference: `nblocks=5` (vs default 1). This enables multi-block drift correction, which tracks drift at 5 depth positions along the probe. This is important for long recordings with non-rigid drift.
