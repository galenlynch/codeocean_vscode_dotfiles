---
name: spikeinterface
description: SpikeInterface framework internals — core abstractions (Recording, Sorting, SortingAnalyzer), the three unit ID spaces, preprocessing chain, postprocessing extensions, quality metrics, curation, phy export, and the spike vector format. Use when working with SI objects, debugging ID mismatches, understanding preprocessing chains, or interpreting quality metrics.
---

# SpikeInterface

SI is the Python framework wrapping spike sorters and providing preprocessing, postprocessing, quality metrics, curation, and export. Local repo: `~/Documents/Code/spikeinterface/`.

## Critical concept: Three ID spaces

This is the single most dangerous source of bugs. See [id-spaces.md](id-spaces.md).

| Space | Example | Where it appears |
|-------|---------|-----------------|
| **unit_id** (= KS cluster_id) | 0, 1, 5, 16, 23... (gaps!) | `sorting.unit_ids`, NWB units table, CSVs |
| **unit_index** | 0, 1, 2, 3, 4... (dense) | `to_spike_vector()["unit_index"]`, phy export, templates array axis |
| **original_cluster_id** | Same as unit_id initially | Phy extractor property, after manual curation may diverge |

## Rules to always remember

1. **Never index positional arrays with unit_id.** `templates[unit_id]` is wrong if there are gaps. Use `sorting.id_to_index(unit_id)` or `sorting.get_unit_template(unit_id=uid)`.

2. **Phy export uses unit_index, not unit_id.** `spike_clusters.npy` contains unit_index values (dense 0..N-1). The mapping back to original unit_ids is via `original_cluster_id` property. See `exporters/to_phy.py:173-178`.

3. **`to_spike_vector()` returns unit_index, not unit_id.** The structured array has fields `("sample_index", "unit_index", "segment_index")`. The `unit_index` is the dense positional index.

4. **After merging/splitting, IDs change.** `remove_units()` doesn't renumber — it removes. `select_units()` keeps the same IDs. But `merge_units()` creates new IDs.

5. **Preprocessing chains are lazy.** `spre.bandpass_filter(recording)` returns a new Recording that computes on-the-fly. Call `.save()` to materialize to binary. The chain is serializable to JSON for reconstruction.

## When to consult reference files

- **ID space bugs, mapping between spaces** — see [id-spaces.md](id-spaces.md)
- **Preprocessing details (filters, CMR, motion)** — see [preprocessing.md](preprocessing.md)
- **Postprocessing extensions and quality metrics** — see [postprocessing.md](postprocessing.md)
- **Curation, de-duplication, auto-merge** — see [curation.md](curation.md)
- **Phy export format and gotchas** — see [phy-export.md](phy-export.md)
