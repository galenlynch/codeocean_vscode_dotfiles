---
name: aind-ephys-pipeline
description: AIND electrophysiology processing pipeline — the full chain from raw Neuropixels recordings through preprocessing, Kilosort4 spike sorting, postprocessing, curation, and NWB export. Covers exact parameters, processing order, known issues, and gotchas. Use when interpreting AIND-sorted data, debugging pipeline outputs, or understanding what happened to the data before it reached your analysis code.
---

# AIND Ephys Pipeline

Nextflow pipeline run on Code Ocean (AWS Batch). Source: [AllenNeuralDynamics/aind-ephys-pipeline](https://github.com/AllenNeuralDynamics/aind-ephys-pipeline).

## Pipeline order

```
Job Dispatch → Preprocessing → Spike Sorting (KS4) → Postprocessing → Curation → Visualization → Results Collection → NWB Export
```

Each probe/shank is processed independently in parallel.

## Rules to always remember

1. **Motion is estimated but NOT applied.** The pipeline computes motion (`dredge_fast` preset) but sets `apply=false`. KS4 does its own drift correction with `nblocks=5`. The motion estimate is only for diagnostics/visualization.

2. **Double CAR/CMR.** Pipeline applies global median CMR, then KS4 applies its own CAR on top (because `skip_kilosort_preprocessing=false`, `do_CAR=true`). The second pass removes residual common mode.

3. **No auto-merging of oversplit units.** KS4's internal merge (ccg_threshold=0.25) is the ONLY merge. The pipeline does not run SI's `auto_merge()`. If KS4 oversplits, you must fix it downstream.

4. **Curation is flags, not removal.** The curation step flags units pass/fail and classifies noise/SUA/MUA, but does NOT remove any units from the output. All units are preserved in the final NWB.

5. **De-duplication threshold is 0.9.** Very conservative — only nearly identical units (>90% spike overlap) are removed.

6. **ISI violation threshold is very permissive.** The curation query allows `isi_violations_ratio < 0.5` — up to 50% ISI violations pass. Don't trust the pass/fail flag blindly for quality.

## When to consult reference files

- **Full preprocessing chain with exact parameters** — see [preprocessing-chain.md](preprocessing-chain.md)
- **KS4 parameters as configured by AIND** — see [sorting-config.md](sorting-config.md)
- **Postprocessing extensions and quality metrics** — see [postprocessing-config.md](postprocessing-config.md)
- **Known issues and gotchas** — see [known-issues.md](known-issues.md)
