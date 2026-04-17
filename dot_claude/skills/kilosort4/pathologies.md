# KS4 Pathologies and Failure Modes

## Oversplitting

**Causes:**
1. Swarmsplitter bimodality threshold is hardcoded at `score < 0.6` (`swarmsplitter.py:114`) — aggressive
2. Bimodality histogram uses hardcoded bins assuming `xproj in [-2, 2]` (`swarmsplitter.py:45`) — unusual morphologies break this
3. High detection thresholds → more separate templates → more initial clusters → more opportunities for bad splits
4. Small `cluster_downsampling` (20) with large datasets → fragmentary clusters
5. Noisy data triggers multiple detections of same spike across templates

**Diagnostics:** Look for pairs of units with very similar templates but split along amplitude, drift, or feature projection axis. Check CCG for refractory violations between the pair — a clean cross-CCG with refractory dip is strong evidence of a real split.

**Mitigations:** Increase `acg_threshold` (allows more refractory violations before splitting), increase `cluster_downsampling`, lower thresholds (but risk false positives).

## Missed spikes

**Causes:**
1. **Dual threshold stacking**: spikes must pass BOTH `Th_universal` (9) AND `Th_learned` (8) — very selective
2. **Template mismatch**: universal templates don't cover all spike morphologies (especially atypical shapes)
3. **max_peels = 100**: matching pursuit stops after 100 iterations per batch regardless of remaining matches
4. **Overlapping spikes**: subtraction errors accumulate over matching pursuit iterations

**Diagnostics:** Compare firing rates to expectations. Look at amplitude distributions for truncation. Check if units appear to "miss" spikes during high-firing epochs.

**Mitigations:** Lower `Th_learned`, set `templates_from_data=True`, increase `max_peels`.

## False merges

**Causes:**
1. Template correlation threshold (`r_thresh=0.5`) too permissive for similar-looking units at different locations
2. Cross-CCG check only examines 1-10ms window (`CCG.py:57`) — longer refractory violations invisible
3. Spike-count-weighted merging (`ns[kk]/(ns[kk]+ns[jj])`) — small genuine units absorbed into large neighbors
4. Temporal misalignment: templates at different peak phases can appear more similar than they are

**Diagnostics:** Check for multi-modal amplitude distributions within a single unit. Look for sudden firing rate changes suggesting two merged sources. Examine spatial extent of unit — unusually large footprint may indicate merge.

## Drift correction artifacts

**Causes:**
1. Drift correction applied BEFORE spike detection — if correction is wrong, downstream is corrupted
2. Over-smoothing (`drift_smoothing=[0.5, 0.5, 0.5]`) can merge separate drift traces
3. Radial basis kernel interpolation introduces small amplitude biases between original and interpolated positions
4. `nblocks > 1` introduces block boundaries where drift estimate can be discontinuous

**Diagnostics:** Plot drift estimate vs time. Look for sudden jumps or implausible drift magnitudes. Compare unit positions across time to see if drift correction created artificial stability or instability.

## Hardcoded thresholds (risk of silent failure)

| Threshold | File:Line | Risk |
|-----------|-----------|------|
| `Cfmax > Th²` (squared threshold) | `template_matching.py:210` | Threshold is squared internally; easy to misunderstand |
| `lam = 20` (regularization) | `template_matching.py:192` | No user control; affects matching pursuit |
| `isort[:nC]` distance filter | `spikedetect.py:230` | Silently drops distant templates without logging |
| `nbins//2` for ACG R00 | `CCG.py:40-41` | ACG bins far from center; misses short refractory |
| `range(1,11)` for CCG Qi | `CCG.py:57` | Only checks 1-10ms window for cross-refractory |
| `xbin[175:225]` bimodality | `swarmsplitter.py:45` | Hardcoded histogram range; fails on unusual projections |
| `erf()` probability sigmoid | `CCG.py:65` | Silent saturation if counts >> lambda |

## No iterative self-consistency (potential GitHub issue)

The split and merge stages run as a single forward pass with no feedback:

1. `clustering_qr.run()` processes each spatial center independently: graph cluster → hierarchical tree → swarmsplitter
2. `merging_function()` runs once globally on the output templates
3. No iteration: merging doesn't re-invoke swarmsplitter, swarmsplitter doesn't see merge results

A bad merge is final. The merged template is a spike-count-weighted average, the spike train is combined, and nobody checks whether the result still looks like a single neuron. Swarmsplitter and the merger also use mismatched criteria — LDA projection bimodality vs template correlation + CCG — so there's no guarantee of consistency.

The snowball mechanism (largest-first, serial absorption, amplitude-blind correlation at 0.5) combined with underpowered CCG checks on small units means the merger simultaneously over-merges large units and leaves small fragments orphaned. See memory `project_ks4_merge_investigation.md` for the full issue draft.

## Double-preprocessing risk

When used via the AIND pipeline or SpikeInterface with `skip_kilosort_preprocessing=False`, KS4 applies its OWN highpass filter (300 Hz) and CAR on data that has ALREADY been highpass-filtered and CMR'd. The second highpass is effectively a no-op, but the second CAR reduces residual common mode — which may be beneficial or may remove signal depending on the data.
