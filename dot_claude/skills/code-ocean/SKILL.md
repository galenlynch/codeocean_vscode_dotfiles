---
name: code-ocean
description: Provides context about Code Ocean capsules, their structure, and how to work with repos that are intended to run on Code Ocean. Use when the current directory is a capsule, or the user asks about Code Ocean.
---

# Code Ocean Compute Capsules

Code Ocean is a cloud-based reproducible computation platform. Capsules are Docker-containerized compute units that run on EC2 instances with attached S3 data assets. They can be run as a **Reproducible Run** (headless, via `run` script), **Run with Parameters** (via App Panel / `config.sh`), or **Cloud Workstation** (interactive IDE session).

Each capsule is a git repo, managed internally by Code Ocean or linked to GitHub.

## Detecting a Capsule

A repo is a Code Ocean capsule if it has a `.codeocean/` directory. Secondary signals: `environment/Dockerfile`, `code/run.sh`, `metadata/metadata.yml`, `.gitignore` entries for `/data/`, `/results/`, `/scratch/`.

## Runtime Filesystem

| Path | Type | Notes |
|------|------|-------|
| `/code` | Read-write | Source code |
| `/data` | **Read-only** | Mounted data assets (one subdirectory per asset) |
| `/results` | Read-write | **Only persistent output** - everything else is lost after a run |
| `/scratch` | Read-write (EFS) | Persistent scratch space across runs |
| `/root` (incl. `/tmp`) | Read-write | **~5 GB limit** - use `/scratch` for large temp files |

## Capsule Conventions

- **Capsules are NOT packages** - `pyproject.toml` has no `[build-system]`, only tooling config
- **uv for dependency management** - `uv sync` locally, `uv pip install` in postInstall
- **Entry point**: `code/run.sh` (bash) → `code/run_capsule.py` (Python `run()` function)
- **Strict linting**: ruff (line length 88, extended rules), NumPy docstring convention
- **GitHub-first**: repos live on GitHub, linked to Code Ocean

## Key Rules

- `/data` is always read-only. Check `.codeocean/datasets.json` for available assets.
- All output must go to `/results`. Files written elsewhere are lost after a Reproducible Run.
- `/tmp` shares the ~5 GB `/root` mount. Large temp files must use `/scratch`.
- Runs are headless - no stdin, no GUI, no interactive prompts.
- Never commit secret values. `.codeocean/secrets.json` contains only references.
- **Do NOT edit `environment/Dockerfile` directly.** It is controlled by the Code Ocean UI and auto-generated from `.codeocean/environment.json`. Direct edits will break the Code Ocean environment editor. To change the environment, edit `environment/postInstall` or use the Code Ocean UI.

## Reference Files

Read these from `${CLAUDE_SKILL_DIR}/` when you need deeper details:
- `codeocean-metadata.md` - `.codeocean/` JSON schemas (datasets, environment, resources, secrets)
- `environment.md` - Dockerfile, postInstall patterns, metadata.yml format
- `pipelines.md` - Capsule vs pipeline differences, data asset types
- `api.md` - Python SDK (`codeocean` package): client setup, attach/detach assets, run capsules, check computations
