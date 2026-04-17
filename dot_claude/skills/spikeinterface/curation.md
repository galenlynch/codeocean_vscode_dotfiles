# SpikeInterface Curation

## De-duplication (`remove_redundant_units`)

Source: `curation/remove_redundant.py`

Compares sorting output with itself to find redundant unit pairs:

1. Uses `compare_two_sorters(sorting, sorting, delta_time=0.4)` — finds coincident spikes within 0.4ms
2. Agreement scores > `agreement_threshold` (0.2) flag potential pairs
3. For each pair, computes `shared = max(coincident/count_i, coincident/count_j)`
4. If `shared > duplicate_threshold` (0.8 in code, **0.9 in AIND pipeline**), units are redundant

**Removal strategies:**
- `"minimum_shift"` (default): keep unit with best peak alignment, break ties by amplitude
- `"highest_amplitude"`: keep larger amplitude unit
- `"max_spikes"`: keep unit with more spikes

**AIND pipeline**: uses threshold 0.9 — very conservative, only nearly identical units removed.

## Auto-labeling (UnitRefine classifiers)

Source: `curation/model_based_curation.py`

```python
# Two-stage classification
noise_labels = scur.auto_label_units(
    analyzer, repo_id="SpikeInterface/UnitRefine_noise_neural_classifier", trust_model=True
)
# Then on neural units only:
sua_mua_labels = scur.auto_label_units(
    analyzer_neural, repo_id="SpikeInterface/UnitRefine_sua_mua_classifier", trust_model=True
)
```

- Downloads pre-trained models from HuggingFace
- Uses quality metrics + template metrics as features
- Returns DataFrame with `prediction` (noise/sua/mua) and `probability` columns
- `trust_model=True` skips model verification

## AIND pipeline curation query

```python
query = "isi_violations_ratio < 0.5 and presence_ratio > 0.8 and amplitude_cutoff < 0.1"
```

This is a **pass/fail flag**, not a removal — all units are kept in the output, just flagged.

**Analysis of thresholds:**
- `isi_violations_ratio < 0.5`: Very permissive. Up to 50% ISI violations (Hill method, 1.5ms threshold). Most real neurons should be well below 0.1.
- `presence_ratio > 0.8`: Moderate. Unit must fire in 80% of 60s bins. Excludes units that appear/disappear (drift, sorting instability).
- `amplitude_cutoff < 0.1`: Strictest criterion. Requires < 10% estimated missing spikes from amplitude distribution truncation.

## What the pipeline does NOT do

- **No auto-merging** of oversplit units. KS4's internal merge (ccg_threshold=0.25) is the only merge that happens.
- **No manual curation integration**. The pipeline outputs flags; downstream analysis must decide what to do with them.
- **No cross-probe de-duplication**. Each probe is processed independently.

## Template metrics dtype bug

The curation capsule has a workaround for template_metrics returning wrong dtypes:
```python
template_metrics_ext.data["metrics"] = template_metrics_ext.data["metrics"].replace("<NA>","NaN").astype("float32")
```
This suggests SI template_metrics can produce `<NA>` values that break downstream pandas operations.
