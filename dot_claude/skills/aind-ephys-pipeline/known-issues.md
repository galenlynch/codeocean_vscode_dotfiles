# AIND Pipeline: Known Issues and Gotchas

## 1. Double preprocessing

**Issue**: KS4 runs with `skip_kilosort_preprocessing=false` and `do_CAR=true`, applying its own highpass filter (300 Hz) and CAR on data that is already highpass-filtered and CMR'd by the pipeline.

**Impact**: The second highpass is a no-op. The second CAR (KS4) on top of CMR (pipeline) may reduce residual common mode noise (beneficial) or reduce signal (harmful for low-amplitude units near common-mode).

**Workaround**: If you suspect double-CAR is causing issues, re-run KS4 with `skip_kilosort_preprocessing=True` or `do_CAR=False`.

## 2. No auto-merging

**Issue**: The pipeline does NOT run SI's `auto_merge()` or any post-hoc merge step. KS4's internal merge (ccg_threshold=0.25, acg_threshold=0.2) is the only merge that occurs.

**Impact**: Oversplit units from KS4 remain oversplit in the final output. This is a deliberate choice (better to oversplit than falsely merge), but means downstream analysis must handle oversplits.

**What to do**: Use SLAy, manual phy curation, or custom merge logic downstream. Our ccg-analysis-capsule handles this.

## 3. Very permissive curation thresholds

**Issue**: The default curation query is:
```
isi_violations_ratio < 0.5 and presence_ratio > 0.8 and amplitude_cutoff < 0.1
```

The ISI violation threshold of 0.5 (50%) is extremely permissive. Most real neurons should have ISI violation ratios well below 0.05.

**Impact**: Many contaminated units pass the default QC flag. Don't rely on the pipeline's pass/fail flag for quality — apply your own, stricter thresholds.

## 4. Motion estimated but not applied

**Issue**: Pipeline default: `motion_correction.compute=true, motion_correction.apply=false`. KS4 handles drift internally with `nblocks=5`.

**Impact**: The pipeline's motion estimate (dredge_fast) and KS4's internal drift correction use different algorithms. They may disagree on drift magnitude/direction. The pipeline's motion is only for visualization.

**For cross-probe analysis**: If comparing spike positions across probes, be aware that each probe's drift is corrected independently by KS4. The pipeline's motion estimate could be used for verification.

## 5. Segment concatenation for KS4

**Issue**: Multi-segment recordings are concatenated before KS4 sorting, then split back after. KS4 has no concept of segment boundaries.

**Impact**: KS4 may detect spikes at concatenation boundaries (if segments have discontinuous data). The `remove_excess_spikes` call after sorting is a partial mitigation.

## 6. Template similarity method discrepancy

**Issue**: Capsule uses `l1`, pipeline passes `cosine_similarity`. De-duplication and similarity-based operations may behave differently depending on how the capsule was invoked.

**Impact**: Usually minor, but could cause subtle differences in which units are flagged as redundant.

## 7. Min drift channels threshold

**Issue**: Drift correction is disabled if `num_channels < 96` (pipeline default) or `< 64` (capsule default when run standalone).

**Impact**: Short probes or probes with many removed bad channels may not get drift correction. This is usually correct (not enough spatial sampling for drift estimation), but worth knowing.

## 8. Template metrics dtype bug

**Issue**: Template metrics can return `<NA>` values that break downstream pandas operations. The curation capsule patches this:
```python
template_metrics_ext.data["metrics"] = template_metrics_ext.data["metrics"].replace("<NA>","NaN").astype("float32")
```

**Impact**: If using template metrics directly from the analyzer without this patch, operations may fail with dtype errors.

## 9. Artifact threshold disabled

**Issue**: `artifact_threshold=null` (becomes `np.inf`). KS4 does not reject any batches based on amplitude.

**Impact**: Large artifacts (e.g., electrical noise bursts) are processed as normal data. KS4 may detect false spikes on artifacts or have artifacts corrupt template estimation. If your data has known artifact periods, consider setting `artifact_threshold` to a finite value.

## 10. De-duplication is very conservative

**Issue**: `duplicate_threshold=0.9` means two units must share >90% of their spikes to be considered duplicates. This catches only near-exact duplicates, not partially overlapping units.

**Impact**: Partially redundant units (sharing 50-80% of spikes) are preserved. This is intentional (avoid false merges) but means the unit count may be inflated.

## 11. SortingAnalyzer can't load recording on reuse

**Issue**: The SortingAnalyzer stored in `postprocessed/*.zarr` contains a preprocessing provenance chain with paths relative to the original pipeline run directory (e.g., `../data/<session>/ecephys/ecephys_compressed/<probe>.zarr`). When the sorted asset is mounted on a different Code Ocean instance (or locally), these paths don't resolve, and `analyzer.recording` raises `ValueError: SortingAnalyzer could not load the recording`.

**Impact**: Any downstream code that needs the recording (SLAy dev, template extraction from raw data, waveform recomputation) fails immediately. `analyzer.has_recording()` returns `False`. The sorting, extensions, and precomputed templates are fine — only the live recording is broken.

**Workaround**: The preprocessing provenance JSON is saved alongside the sorted output at `preprocessed/<recording_name>.json`. The JSON has `relative_paths=true` — SI resolves these against a caller-supplied `base_folder`. On CO, assets are siblings under `/data`, so passing `base_folder=Path("/")` (or `data_dir.parent`) makes `../data/<session>/...` resolve to `/data/<session>/...`:
```python
import json
import spikeinterface as si

prov = json.loads(prov_json_path.read_text())
recording = si.load(prov, base_folder=data_dir.parent)
analyzer.set_temporary_recording(recording)
```
**Important:** use `si.load()` — `si.load_extractor()` was removed in SI 0.104. Don't rewrite paths manually; let SI resolve them via `base_folder`. The `pl-oversplitting-analysis-capsule` has `data_discovery.reconstruct_recording()` that does this automatically.
