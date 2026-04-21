# Capsules vs Pipelines

Pipelines connect multiple capsules into multi-step workflows (Nextflow-based, runs on AWS Batch).

## Key Compatibility Differences

- `/root` does **not** exist in pipelines - use `/data` or `../data`, never `/root/capsule/data`
- Pipeline data may be symlinked from `/tmp` - use `find -L` (follow symlinks) when traversing
- Pipeline capsules ignore capsule-level secrets - a single IAM role is set in Pipeline Settings
- External Data Assets must be copied (no s3fs in Nextflow) - prefer Internal Data Assets for performance

## Data Assets

- **Internal**: Uploaded to Code Ocean's S3 (immutable, cached on EFS, best for reproducibility and pipeline performance)
- **External**: References to external S3 buckets (requires AWS creds, no reproducibility guarantee)
- **Captured Results**: Created from `/results` output of a run (includes provenance tracking)

All mount read-only under `/data/<mount_name>/`.

## Fan-out and parallelism

**Parallelism is configured by the connection between capsules**, not by the capsules themselves. You write per-item output files/folders to `/results` in the upstream capsule, then choose the connection type in the pipeline UI. Nextflow spawns parallel instances based on that choice.

### Connection types — semantics differ by connection origin

The same connection name (**Default**, **Flatten**, **Collect**) means different things depending on whether the upstream is a Data Asset or a Capsule. This is **the single most common source of confusion** when building pipelines.

**Data Asset → Capsule:**

| Type | Behavior |
|------|----------|
| **Default** | Each top-level item (file or folder) in the data asset is passed to a parallel instance of the destination Capsule. Fan-out by item. |
| **Collect** | The entire data asset is passed to a single destination Capsule instance. |

**Capsule → Capsule:**

| Type | Behavior |
|------|----------|
| **Default** | One destination instance per **source instance** (1:1 by instance count, NOT per item). If the upstream ran N parallel instances, downstream runs N times too. |
| **Flatten** | One destination instance per **item** in the upstream's `/results`. Item-level fan-out. |
| **Collect** | One destination instance that sees all items from all upstream instances. |

**Critical takeaway**: Default means completely different things in the two contexts. For a launcher that emits many per-item subfolders and wants each to fan out to its own worker, **you need `Flatten`, not `Default`** on the launcher → worker connection. `Default` would just give you one worker seeing all subfolders.

Items may be passed in a different order than they appear in the source — don't encode order assumptions in items.

### Source Map Path, Target Map Path, wildcards

Each connection has a **Source Map Path** and a **Target Map Path**:

- **Source Map Path** — selects which files/folders from the upstream's `/results` (or from the data asset) feed into this connection. Default preserves directory structure; wildcards change granularity:
  - Empty / default → folder-level at the root of `/results`
  - `probe_*/` → only subfolders matching that glob
  - `**` → recurse into subfolders, pass files at any depth **without preserving directory structure**. The `**` wildcard is paired with `Flatten` — it's what makes per-file fan-out possible in Flatten mode.

- **Target Map Path** — controls **where** in the destination's `/data/` the matched files land. E.g. routing `.zip` files from a data asset into `/data/zip_files/` via a per-connection target path. Default placement is **Collect-vs-Default and Internal-vs-External dependent**:

  | Connection | Internal asset | External asset |
  |---|---|---|
  | Collect | nested: `/data/<asset-name>/...` | **flattened: contents land at `/data/` root** |
  | Default | one top-level item per parallel capsule instance (item-level fan-out; asset type doesn't change this) | same |

  - `<asset-name>` on a Collect+Internal edge is the data asset's `name`/`mount` field, NOT the `.codeocean/datasets.json` mount. Pipeline edges do not inherit `datasets.json` — that file only applies to monolith / Reproducible Runs / Cloud Workstation sessions.
  - Observed 2026-04-20: two External `aind-open-data` assets (raw + sorted) on Collect edges to the same capsule both flattened to `/data/`, causing Nextflow's "input file name collision -- capsule/data" at staging. Fix is either (a) detach one, or (b) set distinct Target Map Paths (e.g. `raw/`, `sorted/`) on each edge so they land at `/data/raw/...` and `/data/sorted/...`.

No capsule code changes are needed to enable fan-out — write structured output to `/results` (e.g. `/results/probe_50213-1/`, `/results/probe_50213-2/`, …) and pick Flatten + `probe_*/` on the connection.

### Gotchas (verbatim from CO's Map Paths docs)

- **Order is not guaranteed.** "Items may be passed in a different order than they appear in the Data Asset" for both Default and Flatten. Don't encode positional assumptions into item names.
- **Unequal fan-out with Default discards extras.** If two Default-connected inputs to a capsule have different item counts, the smaller count wins — "extra items being left out of the computation." If you need full coverage, use Collect for the smaller input.
- **Duplicate filenames across parallel instances → the Collect process never runs.** Any time a fan-out stage feeds a Collect connection, every file or folder an upstream instance writes to `/results` must be namespaced per instance. If N workers each write `/results/<name>` for the same `<name>`, Nextflow aborts the downstream process at staging time with:

  ```
  input file name collision -- There are multiple input files for each of the following file names: capsule/data/<name>
  ```

  The downstream capsule (aggregator) never starts — the pipeline fails before it runs. Collect does not merge, rename, or pick-last; it refuses to stage colliding inputs. Mitigations:
  - Namespace every output by the fan-out unit (e.g. `/results/probe_<name>/summary.csv`, `/results/probe_<name>/pair_figures/*.png`) — this is the pattern used in the `pl-oversplitting-analysis` worker
  - Don't `tee /results/run.log` or otherwise write a fixed top-level filename — CO captures stdout natively, and N workers writing the same log path is this exact collision
  - Enable "Generate indexed folders" on the connection as a fallback, which wraps each source instance's output in a distinct subdir
- **`**` in Source Map Path** flattens directory structure on the way in — files from different subfolders end up side-by-side in the destination `/data/`. If two source subfolders have same-named files, you'll collide. Prefer folder-level fan-out (`probe_*/`) when possible.

## Common patterns

### Launcher → worker (fan-out) → aggregator (collect)

Three-capsule pipeline pattern when you have one input that needs per-item processing and a final summary. A single capsule can play all three roles by dispatching on a subcommand arg passed via each node's Capsule Settings → Arguments field (no need for three separate capsule repos):

1. **Launcher** (1 instance): reads the input data asset, writes one subfolder per item to `/results/probe_<name>/config.json` (or whatever per-item config is needed).
2. **Worker** (N parallel instances): each receives one `probe_*` subfolder from the launcher and its own copy of the full shared data asset. Processes that one item, writes per-item artifacts to `/results/`.
3. **Aggregator** (1 instance): sees all workers' `/results` as subdirectories under `/data/`, produces final combined artifacts.

Connection wiring for this pattern:

| Connection | Type | Source Map Path | Notes |
|---|---|---|---|
| data asset → launcher | Collect | — | Launcher gets the full asset |
| data asset → worker | Collect | — | Each worker gets the full asset (shared) |
| data asset → aggregator | Collect | — | Aggregator gets full asset for post-hoc steps |
| launcher → worker | **Flatten** | `probe_*/` | Fan-out by probe subfolder |
| worker → aggregator | **Collect** | — | Gather all worker outputs to one aggregator |
| aggregator → Results Bucket | — | — | Normal output |

Data-asset-to-capsule connections use **Collect** to avoid splitting the asset across instances — you almost always want each instance to see the complete input data, not a shard.

Make sure each worker's output is namespaced by device/item (e.g. `/results/probe_<name>/`) so the Collect downstream doesn't see filename collisions across parallel instances — otherwise enable "Generate indexed folders" on the worker → aggregator connection.

### Role dispatch with one capsule

Each node in the pipeline UI can set Capsule Settings → **Arguments** — these are forwarded to the capsule's entry script `"$@"`. Use a subparser in `run_capsule.py` so one capsule codebase handles all roles:

```python
sub = parser.add_subparsers(dest="mode")
sub.add_parser("launcher")
sub.add_parser("worker")
sub.add_parser("aggregator")
# + default "monolith" for local/debug
```

Per-node Capsule Settings → Arguments: `launcher` on node 1, `worker` on node 2, `aggregator` on node 3. Same capsule, same image, three different roles. Keeps code sync free.

### Trigger capsule

A "trigger" capsule is a thin capsule whose entire job is kicking off a Pipeline Run programmatically (often via the CO Python SDK against `run_pipeline`). Useful when you want to launch a pipeline from somewhere else (another capsule, a cron, a webhook) without a human clicking "Run".

Note: the dispatcher-via-API pattern is a workaround from before pipelines had good ergonomics. For deterministic fan-out, Nextflow-based pipelines are strictly better — one top-level computation ID, first-class aggregation via Collect, DAG visibility in the UI, pipeline-wide reproducibility. Use a trigger capsule only when you genuinely need conditional/dynamic launching.

## When to choose pipeline vs single capsule

**Single capsule is simpler** — one log, one set of results, no fan-in/fan-out plumbing. Good for serial work or when per-item startup cost dominates the per-item compute time.

**Pipeline wins when**:
- Per-item work exceeds container-startup overhead (~1-3 min per instance)
- Work is trivially parallel across items (no cross-item state)
- Different stages benefit from different machine types (e.g. CPU-heavy preprocessing, then GPU inference)
- You want independent failure isolation (one instance crash doesn't block others)

**Pipeline loses when**:
- Per-item work is small (startup eats the savings)
- Stages share large in-memory state that can't be serialized cleanly
- You want a single consolidated log stream for debugging

### Entry script filename: `run`, not `run.sh`

**Pipelines require the entry script to be named exactly `code/run` (no extension).** `code/run.sh` breaks pipelines — Nextflow won't find an entry and the computation fails almost immediately (often at the ~14 second mark, before any capsule's process logs appear).

Standalone Reproducible Runs accept either `run` or `run.sh`, but since you generally want a capsule to work in both contexts, just always use `run`. It's still a bash script — shebang `#!/usr/bin/env bash` at the top, executable bit set — just no extension.

Symptom of this misconfiguration: pipeline run finishes in ~14 seconds, `exit_code: 1`, `has_results: false`, and the output log only shows Nextflow startup banner with no per-process logs.

### Reproducible Run gate before pipeline use

CO requires each capsule to have **at least one successful Reproducible Run in its history** before it can be used in a pipeline. This is a one-time validation, not per-edit. Code-only edits on a branch don't need a new RR — the pipeline runs off branch HEAD (with `version: null` pinning). Only environment changes (base image, postInstall, `.codeocean/environment.json`) need an image rebuild, not a new RR.

**Implication**: the default RR path matters, even if you intend the pipeline to be the real invocation. Make the default *not* the full expensive pipeline — use a fast-exit **stub mode**:

```python
def run_stub(cfg):
    """No-op to satisfy CO's RR gate without running the full pipeline."""
    cfg.results_dir.mkdir(parents=True, exist_ok=True)
    (cfg.results_dir / "stub.txt").write_text("stub mode\n")
    # exits in seconds, writes a marker, gate passes

# in run():
mode = args.mode or "stub"  # default to stub, pipeline nodes pass explicit role
```

Pipeline nodes set their role via Capsule Settings → Arguments (`launcher`/`worker`/`aggregator`), which overrides the default.

### Path-mapping surprises observed in practice

**CO stages capsule→capsule outputs with extra nesting.** The downstream's `/data/` layout is NOT a direct mirror of the upstream's `/results/`. Confirmed pattern from the MRI pipeline:

- Upstream `bruker_2_nifti` writes to `/results/NIfTI/`.
- Downstream `skull_stripping` sees it at `/data/NIfTI/NIfTI/836656/...` — *double* `NIfTI` because the mount name is `NIfTI` AND the upstream's output subdirectory is also `NIfTI`, and CO preserves both.

So the path the downstream actually sees is typically `/data/<mount-name>/<upstream-subdir>/<files>`, not `/data/<files>` or `/data/<upstream-subdir>/<files>`.

**Input staging uses `/tmp` symlinks.** The Nextflow-generated command on a pipeline worker looks like:
```bash
mkdir -p capsule/data && ln -s $PWD/capsule/data /data
ln -s "/tmp/data/<asset-name>" "capsule/data/<asset-name>"
```
So `/data` is a symlink chain. Python's `Path.rglob` and `Path.glob("**/...")` **do not recurse into symlinked directories by default** (and even 3.13's `recurse_symlinks=True` flag is opt-in). Use `os.walk(followlinks=True)` for any recursive file discovery.

**Target Map Path can flatten or remap.** The MRI skull-strip capsule reads its model as `../data/long_train_unet_3d-V2_nopituitary_26brain.h5` (no subdir) because the pipeline's connection set an explicit Target Map Path that remaps the asset into `/data/` root. By default, without Target Map Path, assets land at `/data/<mount-name>/...`.

### Defensive input discovery

Given the above, capsule code that consumes pipeline-staged inputs should:

- **Use `os.walk(followlinks=True)`** for any recursive search. `Path.glob("**/...")` and `Path.rglob(...)` miss symlinked dirs.
- **Detect inputs by schema, not by path.** Instead of `data_dir.glob("probe_*/config.json")` which assumes a specific wrapping, walk `/data` looking for any file named `config.json` whose content has the expected keys (e.g., `"device_name"` + `"cluster_ids"`). Then the capsule works regardless of how CO wrapped the input.
- **Guard with `len(found) != 1` checks** — if fan-out delivered more or fewer items than expected, fail loud rather than silently picking the wrong one.

Example:

```python
config_paths = []
for root, _, files in os.walk(cfg.data_dir, followlinks=True):
    if "config.json" in files:
        try:
            parsed = json.loads((Path(root) / "config.json").read_text())
        except Exception:
            continue
        if isinstance(parsed, dict) and "device_name" in parsed:
            config_paths.append(Path(root) / "config.json")
if len(config_paths) != 1:
    raise RuntimeError(f"Expected 1 config.json under /data, found {config_paths}")
```

### Mixed-instance-type pipelines

CO ties instance type to base image — a GPU base image forces GPU-class hardware at every node the capsule appears in. For a pipeline where only the worker needs GPU and the launcher/aggregator are CPU-only bookkeeping, this wastes compute and GPU-pool scheduling latency on the short-running nodes.

**Solution: one GitHub repo, two CO capsule entities, different branches.**

| Capsule | Branch | Base image | Role(s) in pipeline |
|---|---|---|---|
| `pl-X` | `main` | GPU (e.g. CUDA-capable) | worker |
| `pl-X-cpu` | `cpu-base` | CPU (slim Ubuntu or CPU-only starter) | launcher, aggregator |

All Python code stays identical. Differences are scoped to:
- `.codeocean/environment.json` (base image)
- `environment/postInstall` (CPU vs CUDA torch wheel channel)
- `.codeocean/resources.json` (smaller instance type)

Periodically `git merge main` into `cpu-base` to sync code changes. Rare merge conflicts because the divergent files are scoped.

### publishDir to Results Bucket flattens by default — map the *directory* to preserve nesting

CO auto-generates publishDir closures on any edge that goes to the Results Bucket. The generated closure looks like:

```groovy
publishDir "$RESULTS_PATH", saveAs: { filename -> new File(filename).getName() }
```

`new File(filename).getName()` returns the basename. If your output is a file glob (e.g. `capsule/results/probe_*/pair_figures/*`), every file from every fan-out instance lands side-by-side at the root of `$RESULTS_PATH`, losing directory structure. Two files with the same basename overwrite each other silently (Nextflow's "last writer wins").

**Note this is the same closure used on fan-out worker → Results Bucket edges and on the final aggregator → Results Bucket edge.** The fan-out case is where collisions bite: namespacing by per-instance subdir (the Collect-collision workaround from above) gets thrown away at publish time.

**Fix at the UI level** — configure the Source Map Path / publish spec to match a **directory**, not a glob of files within it:

| UI input | Generated `output:` path | Result in bucket |
|---|---|---|
| `probe_*/pair_figures/*` | `capsule/results/probe_*/pair_figures/*` | files flattened to root; `probe_X/` namespace lost |
| `probe_*/pair_figures/` | same as above (trailing `/` dropped; sometimes generates a `.matches` regex that never hits, so nothing publishes) | often **nothing** publishes |
| **`probe_*/`** (or `probe_*`) | `capsule/results/probe_*` | each directory published whole; `probe_X/pair_figures/*.png` nesting preserved |

When the matched item is a directory, Nextflow publishDir copies the whole subtree, and the auto-generated `getName()` now returns the *directory* name, so the directory lands intact at `$RESULTS_PATH/probe_X/`.

**Side effect**: publishing the whole `probe_X/` dir also ships any other per-probe artifacts (per-probe CSVs, status JSONs) alongside `pair_figures/`, not just the figures. Usually a feature — keeps the raw per-instance data available in the final result for re-analysis — but worth knowing.

**Verify after changing**: the UI caches results aggressively, so a config change can look like a no-op until you trigger a fresh (uncached) run. If you also need to check the generated Nextflow, pull it from the pipeline's computation artifacts and confirm the `saveAs` regex matches nested files (ends with `/.*`, not just the directory name).

**Bug worth filing**: a UI input of `probe_*/pair_figures/` (with trailing `/`) generates `capsule/results/probe_.*/pair_figures` as the regex — no trailing `/.*`. Groovy's `String.matches()` requires full-string match, so that regex matches the directory path exactly and never any file inside it, so publishing silently drops everything. The "directory vs file-glob" distinction is not surfaced in the UI; users reasonably assume both forms work.

### Common pipeline failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| 14s failure, exit_code=1, no per-process logs | entry script named `run.sh` not `run` | rename |
| Workers spawn but die with "found: []" looking for launcher output | `Path.glob` not recursing symlinks, or hardcoded dir-prefix assumption | switch to `os.walk(followlinks=True)` + schema-based detection |
| Aggregator never runs; pipeline fails with `input file name collision -- capsule/data/<name>` | fan-out workers wrote the same top-level name to `/results` (bare file or folder) — Collect refuses to stage duplicates | namespace every worker output per fan-out unit (`/results/probe_<name>/...`); never `tee` a fixed log path |
| Files published to Results Bucket but flattened (no per-instance subdirs) | UI publish spec was a file glob like `probe_*/pair_figures/*`; auto-generated `saveAs` closure returns `getName()` which strips directories | switch the UI publish spec to the *directory* (`probe_*/`) — whole subtree publishes with structure intact |
| UI publish spec `probe_*/pair_figures/` with trailing `/` — nothing lands in bucket | CO generates regex `capsule/results/probe_.*/pair_figures` with no trailing `/.*`; Groovy `.matches()` requires full match, fails on every file | add `*` for file-glob form, or switch to directory form `probe_*/` — file scopes the match, directory publishes whole tree |
| Aggregator sees no worker outputs (but ran) | `Path.glob` not recursing symlinks, or hardcoded dir-prefix assumption | switch to `os.walk(followlinks=True)` + schema-based detection |
| `FileNotFoundError` discovering data assets under `/data` | `Path.rglob` skipping the `/tmp` symlink | `os.walk(followlinks=True)` |
| Image build fails on code-server tarball extraction | base image already has code-server, UI-added `"vscode": {}` with packages triggers duplicate install | leave `"vscode": {}` empty in `.codeocean/environment.json` |
| Pipeline gate rejects capsule | no successful RR in history | make default mode a fast-exit stub, click RR once, gate passes |
| Extra items discarded on Default fan-in | unequal-count inputs | use Collect for the smaller input |

## Testing the pipeline

Pipelines require released capsule versions. Iterate via:

1. Get the capsule working as a monolith first (`run_capsule.py monolith`) — validates the core logic
2. Factor into launcher/worker/aggregator entry points, test each individually with hand-constructed `/data` layouts locally
3. Release a capsule version, build the pipeline in CO UI
4. Test the pipeline against a small input (e.g. 1 probe) to verify dispatch and IO contracts
5. Scale up to full input

Debugging a failing pipeline: check the per-node logs individually in the CO pipeline UI. Workers in Flatten fan-out each have their own log; look at the one that failed.
