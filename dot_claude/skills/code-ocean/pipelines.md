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
