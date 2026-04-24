# .codeocean/ Metadata Files

The `.codeocean/` directory is **written by the Code Ocean UI** and round-tripped on every save. Anything the UI doesn't explicitly model will be stripped or re-added on the next UI edit. Two practical rules:

1. **Tabs, not spaces, for indentation.** CO's JSON writer uses tabs.
2. **No trailing newline on any file in `.codeocean/`.** CO strips trailing newlines. If your editor adds one, git will show spurious diffs every time the UI saves.

Verify after editing:
```bash
tail -c 3 .codeocean/environment.json | od -c     # expect "}" at end, no \n
xxd .codeocean/resources.json | tail -2            # inspect raw bytes
```

Python one-liner to strip a trailing newline cleanly:
```bash
python3 -c "p='.codeocean/X.json'; data=open(p,'rb').read().rstrip(b'\n'); open(p,'wb').write(data)"
```

## datasets.json
Lists attached data assets with UUIDs and short mount names. Each asset appears as a subdirectory under `/data/` using its `mount` name.
```json
{
  "version": 1,
  "attached_datasets": [
    {"id": "uuid-here", "mount": "sorted"}
  ]
}
```
At runtime (interactive / monolith RR): `/data/sorted/`.

**Short mount names apply to pipeline capsules only.** Pipeline capsules get their data from upstream nodes and benefit from stable short names (`sorted`, `ccg`, `raw`, `nwb`) independent of the source asset's full AIND ID. For standalone capsules — interactive / Cloud Workstation / Reproducible Run — leave the mount blank and `/data/` will use the full AIND asset name; renaming there has no upside and obscures which asset is actually attached.

**Pipeline caveat**: for pipeline runs, the data-asset mount names come from the **pipeline's** data-asset attachments (set in the pipeline UI), not from the capsule's `datasets.json`. Rename in both places if you want matching paths.

## environment.json
Defines the base Docker image, package installers, and build options. The `environment/Dockerfile` is auto-generated from this by Code Ocean's UI — **do not edit the Dockerfile manually** (CO overwrites on next save).
- `base_image`: Starter environment (Python, R, MATLAB, Ubuntu, tensorflow GPU, etc.)
- `post_install`: Whether `environment/postInstall` runs during build
- `installers.apt.packages`: System packages (can pin versions)
- `installers.vscode.packages`: Code-server extensions (optional; see quirk below)
- `options.mount_secrets`: Whether secrets are available during build (for private repos)
- `env_variables`: Environment variables set in the container

### The `"vscode": {}` quirk
The CO UI **always writes an `"installers.vscode"` key**, even if empty. If you remove the key by hand, the next UI save adds it back. Accept it and leave it empty:
```json
"installers": {
    "apt": { ... },
    "vscode": {}
}
```

If the `vscode` block has `packages` with specific extensions AND a `version`, CO generates a Dockerfile that downloads a full `code-server` tarball and extracts it. On base images that *already* include code-server (e.g. `codeocean/code-server-python-extensions-pack:v5.0`), the tarball extraction conflicts and fails:
```
process "... tar -xvf code-server.tar.gz" did not complete successfully: exit code: 1
```
Fix: keep `vscode` as an empty object `{}` — CO won't emit the tarball-extract steps unless extensions or a version are specified.

## resources.json
Compute resource class (determines CPU, RAM, GPU). Two schemas in use:
```json
{"version": 1, "resource_class": "medium"}
```
vs. explicit instance type:
```json
{
    "version": 1,
    "resource_class": "dedicated",
    "machine_config": {"instance_type": "c7a.xlarge"}
}
```
Use the explicit `machine_config.instance_type` form when you need a specific AWS family (e.g. `c7a.xlarge` AMD Genoa for CPU, `g6.4xlarge` Sapphire Rapids + L4 for GPU). The `resource_class` string is less flexible.

**Instance type is tied to base image**. A GPU base image forces a GPU-class instance; a CPU base image forces CPU-only. To mix CPU and GPU nodes in a pipeline, use separate capsules / branches with different base images (see `pipelines.md` → "Mixed-instance-type pipelines").

## secrets.json
Declares secrets attached to the capsule. Types: `general` (env vars), `aws` (IAM credentials). Available as environment variables at runtime and optionally during build. Only references to CO-managed secrets are stored here — never paste actual secret values.

## Things CO's UI controls that you shouldn't edit by hand
- `environment/Dockerfile` — auto-generated from `environment.json`. Edits get overwritten.
- The `# hash:sha256:...` comment on line 1 of `Dockerfile` — CO recomputes it from the rest of the file. If you edit the Dockerfile manually, recompute with:
  ```bash
  tail -n +2 environment/Dockerfile | sha256sum | cut -d' ' -f1
  ```
  CO will still regenerate on next UI save.

## Things the UI tends to silently round-trip
Even with no user-visible change, a UI save may:
- Strip trailing newlines from `.codeocean/*` files.
- Re-add an empty `"installers.vscode": {}` block to `environment.json`.
- Reorder keys or re-emit `"version"` variants.

Don't fight these — match the UI's expected shape (tabs, no trailing newline, empty `vscode: {}`) so `git diff` stays clean.
