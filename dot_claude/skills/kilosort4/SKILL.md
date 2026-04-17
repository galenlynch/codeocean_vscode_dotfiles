---
name: kilosort4
description: Kilosort 4 spike sorting internals — pipeline stages, template matching, drift correction, clustering, merge/split logic, parameters, output format, and known pathologies. Use when working with KS4 output, debugging sorting results, interpreting cluster IDs, or tuning KS4 parameters.
---

# Kilosort 4

KS4 is a GPU-accelerated Python spike sorter using graph-based clustering. Local repo: `~/Documents/Code/ccg-analysis-monorepo/Kilosort/`.

## Pipeline stages (in order)

1. **Preprocessing** (`compute_preprocessing`): highpass filter + whitening matrix computation
2. **Drift correction** (`compute_drift_correction`): detect spikes on universal templates, estimate vertical drift per batch
3. **Spike detection** (`detect_spikes`): two-stage — universal template detection, then learned template extraction via matching pursuit
4. **Clustering** (`cluster_spikes`): graph-based QR clustering with hierarchical merge/split
5. **Save** (`save_sorting`): output to phy-compatible format with quality labels

## Thresholds that matter

| Parameter | Default | What it controls |
|-----------|---------|-----------------|
| `Th_universal` | 9 | Detection threshold for universal templates (squared internally: `Cf > Th²`) |
| `Th_learned` | 8 | Detection threshold for learned templates |
| `acg_threshold` | 0.2 | ACG refractory violation fraction; < this = "good" |
| `ccg_threshold` | 0.25 | Cross-CCG threshold for merge candidates |
| `duplicate_spike_ms` | 0.25 | Same-cluster dedup window |
| `nblocks` | 1 (KS4 default) / 5 (AIND) | 0 = no drift correction, >1 = multi-block |
| `cluster_downsampling` | 20 | Spikes per clustering center; higher = faster but less precise |
| `max_channel_distance` | null | If set, filters out distant template matches |

## How detection works

**Universal templates** (`spikedetect.py`): Multi-scale Gaussian envelope templates on a spatial grid (`dmin` x `dminx`). 5 sizes x 5 scales = 25 templates. If `templates_from_data=True`, PCA + KMeans on high-SNR snippets to learn data-driven shapes.

**Learned templates** (`template_matching.py`): Matching pursuit deconvolution with `max_peels=100` iterations per batch. Each iteration finds the best-matching template, subtracts it, repeats. Spikes must pass BOTH thresholds (universal then learned) — this is very selective.

## How clustering works

**Graph-based QR** (`clustering_qr.py`): Faiss nearest-neighbor search on downsampled spike features. KMeans++ initialization, iterative assignment with soft membership via sparse tensors. ~200 iterations.

**Hierarchical merge** (`hierarchical.py`): Dendrogram built from connection strength between adjacent clusters. Modularity-based: `crat = cc/cneg`.

**Split logic** (`swarmsplitter.py`): Sequential criteria — (1) global modularity < 0.2, (2) refractoriness via CCG, (3) bimodality via LDA (score < 0.6), (4) local modularity > 0.15. Splits prevented if parent cluster is refractory (R12 < 0.1).

**Final merge** (`template_matching.py:merging_function`): Template correlation + CCG check. Refractory clusters merge if correlation > 0.5. Cross-CCG must satisfy R12 < `ccg_threshold` AND Q12 < 0.05. Merge weighted by spike count — larger clusters dominate.

## Cluster ID assignment

- IDs assigned sequentially based on dendrogram leaves, offset by processing order through x,y template centers
- **CAN have gaps** if clusters merge or are removed
- IDs are NOT stable across runs (depend on random seed `cluster_init_seed`, default 5)
- Processing order: spatial grid centers in x,y order (`clustering_qr.py:451-452`)
- Mechanism: `clu[igood] = iclust + nmax` where nmax is cumulative offset per center

## Output files (phy format)

```
spike_times.npy          # (n_spikes,) int64 — sample times
spike_clusters.npy       # (n_spikes,) int32 — cluster ID per spike
spike_templates.npy      # (n_spikes,) int32 — detection template (obsolete copy)
templates.npy            # (n_clusters, n_samples, n_channels) — reconstructed templates
amplitudes.npy           # (n_spikes,) float32 — spike amplitude
cluster_KSLabel.tsv      # "good" or "mua" per cluster
cluster_ContamPct.tsv    # contamination rate x 100
```

## Known pathologies

See [pathologies.md](pathologies.md) for full analysis with mitigation strategies.

**Key failure modes:**
- **Oversplitting**: aggressive swarmsplitter bimodality (hardcoded score < 0.6), noisy data, high detection thresholds
- **Missed spikes**: dual threshold stacking (universal + learned), low max_peels for overlapping spikes
- **False merges**: high template correlation threshold (0.5), CCG check window only 1-10ms, spike-count-weighted merging loses small units
- **Drift overcorrection**: applied pre-detection, can create false merges if over-smoothed

## When to consult reference files

- **Pathologies, failure modes, hardcoded thresholds** — see [pathologies.md](pathologies.md)
- **Parameter tuning guide** — see [parameters.md](parameters.md)
