# .codeocean/ Metadata Files

## datasets.json
Lists attached data assets with UUIDs and mount names. Each asset appears as a subdirectory under `/data/` using its `mount` name.
```json
{
  "version": 1,
  "attached_datasets": [
    {"id": "uuid-here", "mount": "my_dataset_name"}
  ]
}
```
At runtime: `/data/my_dataset_name/`.

## environment.json
Defines the base Docker image, package installers, and build options. The `environment/Dockerfile` is auto-generated from this by Code Ocean's UI.
- `base_image`: Starter environment (Python, R, MATLAB, Ubuntu, etc.)
- `post_install`: Whether `environment/postInstall` runs during build
- `installers.apt.packages`: System packages
- `options.mount_secrets`: Whether secrets are available during build (for private repos)

## resources.json
Compute resource class (determines CPU, RAM, GPU).
```json
{"version": 1, "resource_class": "medium"}
```

## secrets.json
Declares secrets attached to the capsule. Types: `general` (env vars), `aws` (IAM credentials). Available as environment variables at runtime and optionally during build.
