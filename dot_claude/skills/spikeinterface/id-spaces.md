# SpikeInterface ID Spaces

## The three ID spaces

### 1. unit_id (= KS cluster_id, zarr unit_id)

- The "name" of a unit. Can be int or str. Has gaps (e.g., 0, 1, 5, 16, 23...).
- Stored in `sorting.unit_ids` (numpy array).
- Used by all SI public API: `get_unit_spike_train(unit_id=...)`, `get_template(unit_id=...)`.
- When loading KS4 output via `KiloSortSortingExtractor`, unit_ids ARE the KS cluster_ids from `spike_clusters.npy`.
- Preserved in NWB units table, CSV outputs, `detected_pairs.csv`.

### 2. unit_index (dense positional index)

- Dense 0..N-1 index into arrays. `unit_index = sorting.id_to_index(unit_id)`.
- Used in:
  - `to_spike_vector()["unit_index"]` — the spike vector's unit field
  - Phy export: `spike_clusters.npy`, `spike_templates.npy` (written as unit_index!)
  - `templates.npy` position axis in phy export
  - SLAy similarity matrices
  - Any numpy array indexed by unit position

### 3. original_cluster_id (phy property)

- When loading FROM phy (after manual curation), the `PhySortingExtractor` reads `cluster_id` from TSV files and stores as `original_cluster_id` property.
- After phy curation, `spike_clusters.npy` values may not match original KS cluster IDs (phy can renumber on merge/split).
- If `si_unit_id` column exists in phy cluster_info, it's used as the SI unit_id.

## The dangerous pattern

```python
# WRONG — silent data corruption if unit_ids have gaps
template = templates_array[ks_unit_id]
similarity = slay_matrix[ks_unit_id, :]

# For units below the first gap, index == id, so it appears to work.
# Above the first gap, every lookup is off by cumulative gap count.
```

## Safe patterns

```python
# Use SI API — handles conversion internally
spike_train = sorting.get_unit_spike_train(unit_id=ks_id)
template = analyzer.get_template(unit_id=ks_id)

# Use dicts keyed by unit_id, not positional arrays
templates_dict = {uid: analyzer.get_template(unit_id=uid) for uid in sorting.unit_ids}

# When you need the index explicitly
idx = sorting.id_to_index(ks_id)
template = templates_array[idx]

# Building a mapping
id_to_idx = {uid: i for i, uid in enumerate(sorting.unit_ids)}
```

## Known victims of this bug

- `rebuild_cluster_group.py` (SLAy staging) — compared phy cluster IDs against NWB ks_unit_ids without conversion
- Earlier `oversplit_diagnostic_panel.py` — indexed SI template arrays by ks_unit_id
- SLAy AE similarity results may be invalid due to wrong unit labels

## Phy export ID mapping

In `exporters/to_phy.py:173-178`:
```python
all_spikes_seg0 = sorting.to_spike_vector(concatenated=False)[0]
spike_times = all_spikes_seg0["sample_index"]
spike_labels = all_spikes_seg0["unit_index"]  # <-- unit_INDEX, not unit_id!
np.save("spike_clusters.npy", spike_labels)   # phy sees these as "cluster IDs"
```

So phy's cluster IDs are actually SI's unit_indices. The `original_cluster_id` property maps back.

## After operations that change units

| Operation | Effect on unit_ids |
|-----------|-------------------|
| `sorting.remove_units(ids)` | Removes those IDs. Remaining IDs unchanged. unit_indices change! |
| `sorting.select_units(ids)` | Keeps those IDs. unit_indices change! |
| `analyzer.merge_units(groups)` | Creates new unit_ids (typically max_id + 1). Old IDs gone. |
| `sc.remove_redundant_units()` | Returns new sorting with subset of original unit_ids |
| `sorting.remove_empty_units()` | Removes units with 0 spikes |
