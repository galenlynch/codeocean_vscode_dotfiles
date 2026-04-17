# Code Ocean Python SDK (`codeocean`)

- PyPI: `codeocean` — requires Python ≥ 3.11 and Code Ocean platform ≥ 2.19
- Source: https://github.com/codeocean/codeocean-sdk-python
- User guide: https://docs.codeocean.com/user-guide/code-ocean-api

Install with `pip install -U codeocean` or `uv add codeocean`.

## Client

```python
from codeocean import CodeOcean

client = CodeOcean(
    domain="https://codeocean.allenneuraldynamics.org",  # AIND deployment
    token="cop_...",                                     # from Account → Access Tokens
    retries=3,                                           # int or urllib3 Retry
    agent_id="Claude Code",                              # optional, for AI-agent attribution
)
```

- Auth: HTTP Basic — token is the username, password is empty. The SDK handles this.
- Tokens start with `cop_` and are only shown once on creation.
- The client exposes sub-APIs as attributes: `client.capsules`, `client.pipelines`, `client.computations`, `client.data_assets`, `client.custom_metadata`.
- The top-level package re-exports only `CodeOcean` and `Error`. All other types live under `codeocean.computation`, `codeocean.data_asset`, `codeocean.capsule`, `codeocean.pipeline`, `codeocean.components`, `codeocean.custom_metadata`.

## Error handling

Every non-2xx response raises `codeocean.Error` (a `requests.HTTPError` wrapper):

```python
from codeocean import Error

try:
    client.data_assets.get_data_asset("bad-id")
except Error as e:
    e.status_code   # int, e.g. 404
    e.message       # parsed from response body if JSON
    e.data          # full JSON body or None
    e.http_err      # original requests.HTTPError
```

Status codes: `400` bad request, `401` no/invalid token, `403` forbidden, `404` not found, `429` rate limited (back off and retry), `5xx` server error. The `retries=` parameter on `CodeOcean(...)` handles transient errors automatically.

## Attached vs. runtime data assets — two different types

This trips people up: there are **two** different "data asset params" classes.

| Use case | Class | Where imported from |
|---|---|---|
| Attach/detach asset on a capsule or workstation (persistent) | `DataAssetAttachParams` | `codeocean.data_asset` |
| Attach an asset for a single run (passed in `RunParams`) | `DataAssetsRunParam` | `codeocean.computation` |

Both have `id` and optional `mount`, but they hit different endpoints. Use `DataAssetsRunParam` when constructing `RunParams`; use `DataAssetAttachParams` when calling `client.capsules.attach_data_assets(...)`.

## Data assets

### Get / search

```python
from codeocean.data_asset import DataAssetSearchParams, DataAssetType

asset = client.data_assets.get_data_asset("uuid")

# Single page
res = client.data_assets.search_data_assets(
    DataAssetSearchParams(query="ecephys_786867", type=DataAssetType.Dataset, limit=100)
)
res.has_more; res.next_token; res.results  # list[DataAsset]

# Auto-paginated iterator
for asset in client.data_assets.search_data_assets_iterator(
    DataAssetSearchParams(query="tag:genomics")
):
    print(asset.id, asset.name)
```

Search query syntax (from the SDK docstring): `field:value` pairs joined by AND across different fields, OR when the same field repeats, quotes for exact phrases. Valid fields: `name`, `tag`, `run_script`, `commit_id`, `contained_data_id`. No wildcards, not case-sensitive.

### Attach to a capsule (persistent mount across runs)

```python
from codeocean.data_asset import DataAssetAttachParams

results = client.capsules.attach_data_assets(
    capsule_id="...",
    attach_params=[DataAssetAttachParams(id="uuid", mount="my-mount")],
)
# results: list[DataAssetAttachResults] — check .ready, .mount_state

client.capsules.detach_data_assets(capsule_id="...", data_assets=["uuid"])
```

Pipelines use the same interface via `client.pipelines.attach_data_assets(...)`.

### Create a new data asset

Four source types (exactly one must be set on `Source`):

- `AWSS3Source(bucket, prefix=..., keep_on_external_storage=..., public=...)` — import from S3
- `GCPCloudStorageSource(bucket, ...)` — import from GCP
- `ComputationSource(id, path=...)` — capture computation results
- `CloudWorkstationSource(id, path, run_script=...)` — capture from a live workstation

```python
from codeocean.data_asset import (
    DataAssetParams, Source, ComputationSource, AWSS3Source,
    Target, AWSS3Target,
)

# Capture results as an internal asset
params = DataAssetParams(
    name="sorted_2026-04-16",
    tags=["derived", "ecephys", "subject-123"],
    mount="sorted",
    source=Source(computation=ComputationSource(id=computation.id)),
    custom_metadata={"data level": "derived", "subject id": "123"},
)

# Capture as an external asset (S3) — pass target=
params = DataAssetParams(
    name="...", tags=[...], mount="...",
    source=Source(computation=ComputationSource(id=computation.id)),
    target=Target(aws=AWSS3Target(bucket="aind-open-data", prefix=...)),
)

asset = client.data_assets.create_data_asset(params)
asset = client.data_assets.wait_until_ready(asset, polling_interval=10, timeout=300)
```

`create_data_asset` returns immediately with a draft asset — actual creation is async. `wait_until_ready` polls `get_data_asset` until state is `Ready` or `Failed`. Minimum `polling_interval` is 5 seconds (SDK-enforced).

### Other data asset ops

```python
from codeocean.components import Permissions, EveryoneRole

client.data_assets.update_permissions(
    data_asset_id="...",
    permissions=Permissions(everyone=EveryoneRole.Viewer, share_assets=True),
)
client.data_assets.update_metadata(data_asset_id, DataAssetUpdateParams(tags=[...]))
client.data_assets.archive_data_asset(data_asset_id, archive=True)
client.data_assets.delete_data_asset(data_asset_id)
client.data_assets.list_data_asset_files(data_asset_id, path="")  # Folder
client.data_assets.get_data_asset_file_urls(data_asset_id, path="foo.nwb")  # FileURLs
```

Admin-only: `transfer_data_asset(id, TransferDataParams(target=..., force=...))`.

## Capsules and pipelines

The `Capsules` and `Pipelines` clients have nearly identical surfaces (pipelines are implemented internally by reusing `Capsules` with a different route). Both support:

```python
client.capsules.get_capsule(capsule_id)
client.capsules.get_capsule_app_panel(capsule_id, version=None)  # AppPanel with params
client.capsules.list_computations(capsule_id)
client.capsules.attach_data_assets(capsule_id, [DataAssetAttachParams(...)])
client.capsules.detach_data_assets(capsule_id, [asset_id])
client.capsules.get_permissions(capsule_id)
client.capsules.update_permissions(capsule_id, Permissions(...))
client.capsules.archive_capsule(capsule_id, archive=True)
client.capsules.search_capsules(CapsuleSearchParams(...))
client.capsules.search_capsules_iterator(CapsuleSearchParams(...))
```

Same methods exist under `client.pipelines` with `pipeline_id` args (e.g. `get_pipeline`, `search_pipelines`).

## Running computations

`client.computations.run_capsule(RunParams)` handles both capsule and pipeline runs — set either `capsule_id` or `pipeline_id`. `run_pipeline` is an alias.

### Capsule run

```python
from codeocean.computation import RunParams, DataAssetsRunParam, NamedRunParam

computation = client.computations.run_capsule(
    RunParams(
        capsule_id="...",
        version=3,                             # optional: pinned release
        data_assets=[DataAssetsRunParam(id="uuid", mount="raw")],
        parameters=["--flag", "value"],        # positional, passed to run
        # OR:
        named_parameters=[NamedRunParam(param_name="alpha", value="0.5")],
    )
)
```

### Pipeline run with per-process parameters

This is the pattern used when a pipeline has multiple processes each taking their own positional args (see `AllenNeuralDynamics/spikesorting-trigger-capsule/code/main.py` for a production example):

```python
from codeocean.computation import RunParams, DataAssetsRunParam, PipelineProcessParams

run_params = RunParams(
    pipeline_id="...",
    version=2,
    data_assets=[DataAssetsRunParam(id="uuid", mount="ecephys_session")],
    processes=[
        PipelineProcessParams(
            name="job_dispatch",                  # name from main.nf
            parameters=["true", "false", "30"],   # positional, all strings
        ),
        PipelineProcessParams(
            name="spike_sort",
            named_parameters=[NamedRunParam(param_name="sorter", value="ks4")],
        ),
    ],
    nextflow_profile="prod",
)
computation = client.computations.run_pipeline(run_params)
```

`parameters` is an **ordered list of strings** — order must match the process's `argparse` positional args. Adding a new positional without updating every caller (including Code Ocean UI panels) shifts everything — a real footgun.

### Wait for completion

```python
done = client.computations.wait_until_completed(
    computation, polling_interval=60, timeout=7 * 24 * 3600,
)
```

Minimum `polling_interval` is 5 seconds. Returns when state reaches `Completed` or `Failed` (note: **not** the same as success — check `end_status`).

### Checking success

A run is only successful if **all** of these hold:

```python
from codeocean.computation import ComputationState, ComputationEndStatus

succeeded = (
    done.state == ComputationState.Completed
    and done.end_status == ComputationEndStatus.Succeeded
    and (done.exit_code is None or done.exit_code == 0)
)
```

`ComputationState` values: `Initializing`, `Running`, `Finalizing`, `Completed`, `Failed`.
`ComputationEndStatus` values: `Succeeded`, `Failed`, `Stopped`.

### Computation results & files

```python
client.computations.list_computation_results(computation_id, path="")       # Folder
client.computations.get_result_file_urls(computation_id, path="foo.nwb")    # view + download URLs
client.computations.rename_computation(computation_id, name="...")
client.computations.delete_computation(computation_id)
```

### Cloud workstation only — attach/detach at runtime

```python
client.computations.attach_data_assets(computation_id, [DataAssetAttachParams(...)])
client.computations.detach_data_assets(computation_id, [asset_id])
```

These only work on cloud workstation computations, not regular runs.

## Common gotchas

- **Two asset param classes.** `DataAssetsRunParam` (runtime, in `RunParams`) vs `DataAssetAttachParams` (persistent capsule attachment). Easy to confuse.
- **`parameters` are all strings.** The SDK does not coerce — `str(p)` everything, including ints/bools/floats, before passing.
- **Positional parameter order.** For processes with positional args, inserting a new arg in the middle silently shifts every downstream value. Prefer `named_parameters` (`NamedRunParam`) where practical, or always append at the end.
- **`wait_until_completed` returns on `Failed` too.** Always inspect `end_status` and `exit_code` after, not just `state`.
- **`create_data_asset` is async.** The returned `DataAsset` is in `Draft` state until `wait_until_ready` confirms `Ready`.
- **Polling interval ≥ 5s.** SDK raises `ValueError` below that.
- **AIND domain.** `https://codeocean.allenneuraldynamics.org` — not `alleninstitute.org`.
- **Import paths.** Top-level package only re-exports `CodeOcean` and `Error`. Types live in submodules. Some repos use `from codeocean.client import CodeOcean as CodeOceanClient` — equivalent, just more explicit.

## MCP vs SDK

The `mcp__codeocean__*` tools (from `codeocean-mcp-server`) provide live API access in Claude Code conversations — use those for interactive lookups. The Python SDK is for writing code that runs on Code Ocean or orchestrates it from scripts. They hit the same API but serve different contexts.
