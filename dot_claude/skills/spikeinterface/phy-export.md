# SpikeInterface Phy Export

Source: `exporters/to_phy.py`

## Critical: Phy uses unit_index, not unit_id

```python
# to_phy.py:173-178
all_spikes_seg0 = sorting.to_spike_vector(concatenated=False)[0]
spike_times = all_spikes_seg0["sample_index"]
spike_labels = all_spikes_seg0["unit_index"]  # DENSE 0..N-1
np.save("spike_times.npy", spike_times[:, np.newaxis])
np.save("spike_templates.npy", spike_labels[:, np.newaxis])
np.save("spike_clusters.npy", spike_labels[:, np.newaxis])
```

When you load a phy folder back into SI, the `PhySortingExtractor` reads `spike_clusters.npy` and treats those values as cluster IDs. The `original_cluster_id` property maps back to the original KS unit_ids.

## Files written by export_to_phy

```
output_folder/
├── params.py                # dat_path, n_channels_dat, dtype, offset, sample_rate
├── recording.dat            # binary recording (if copy_binary=True)
├── spike_times.npy          # (n_spikes, 1) int64 — sample indices
├── spike_templates.npy      # (n_spikes, 1) int32 — unit_index (same as spike_clusters)
├── spike_clusters.npy       # (n_spikes, 1) int32 — unit_index (dense 0..N-1)
├── templates.npy            # (n_units, n_samples, max_sparse_chans) float64
├── templates_ind.npy        # (n_units, max_sparse_chans) int64 — channel indices per template
├── similar_templates.npy    # (n_units, n_units) float32 — pairwise similarity
├── channel_map.npy          # (n_channels,) — channel indices
├── channel_positions.npy    # (n_channels, 2) — x,y positions
├── pc_features.npy          # (n_spikes, n_components, n_sparse_chans) (if computed)
├── pc_feature_ind.npy       # (n_units, n_sparse_chans) (if computed)
├── amplitudes.npy           # (n_spikes, 1) (if computed)
├── cluster_*.tsv            # quality metrics, template metrics as TSV
└── channel_groups.npy       # channel group assignments
```

## Key differences from KS4 native phy output

| Aspect | KS4 native | SI export |
|--------|-----------|-----------|
| spike_clusters values | KS cluster IDs (with gaps) | Dense unit_index (0..N-1) |
| templates array | All channels | Sparse (max_sparse_chans) |
| templates_ind | Not present | Maps sparse channels to full channel set |
| amplitudes | KS internal amplitude | SI-computed spike amplitudes |
| Quality TSVs | cluster_KSLabel, cluster_ContamPct | SI quality metrics, template metrics |

## Loading phy back into SI

```python
sorting = si.read_phy(folder_path="phy_output/")
# unit_ids will be the cluster IDs from spike_clusters.npy (which are unit_indices!)
# original_cluster_id property maps back to original IDs
```

After manual curation in phy (merging/splitting), the cluster IDs in `spike_clusters.npy` are modified by phy directly. The `cluster_group.tsv` file tracks manual labels (good/mua/noise).

## Gotcha: single-segment only

`export_to_phy` only works with single-segment recordings. Multi-segment recordings must be concatenated first.
