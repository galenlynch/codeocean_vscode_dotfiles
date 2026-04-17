---
name: pydantic-settings-cli
description: Patterns and gotchas for building CLI apps with pydantic-settings. Covers CliApp.run/run_subcommand, subcommands, error handling, BaseSettings configuration, env vars, dotenv, Field descriptions, and type coercion. Use when writing or debugging pydantic-settings CLI code.
---

# Pydantic Settings CLI

Reference for `pydantic-settings>=2.0` CLI apps.

## When to consult reference files

- **ConfigDict options** (all CLI/env/secrets options with defaults): see `reference/configdict-options.md`
- **Worked examples** (multi-subcommand, progress bars, env fallbacks, config objects): see `reference/examples.md`

## Critical Gotcha: Subcommand Dispatch

`CliApp.run()` alone does **NOT** dispatch to subcommand `cli_cmd()` methods. You **MUST** call `CliApp.run_subcommand()` afterward. Without it, the CLI silently exits code 0 with no output.

```python
# WRONG — silently does nothing:
CliApp.run(CLI)

# CORRECT:
model = CliApp.run(CLI)
CliApp.run_subcommand(model)
```

## Preferred Defaults

Always use these ConfigDict options on the root CLI class unless the project overrides:
- `cli_kebab_case=True` — flags use `--kebab-case` not `--snake_case`
- `cli_implicit_flags=True` — booleans accept `--verbose` / `--no-verbose` (not `--verbose True`)

## Minimal Working Pattern

```python
from pydantic import BaseModel, ConfigDict, Field
from pydantic_settings import BaseSettings, CliApp, CliSubCommand

class SubCmd(BaseModel):
    name: str = Field(description="shown in --help")
    verbose: bool = False  # with implicit_flags: --verbose / --no-verbose
    def cli_cmd(self) -> None:
        print(self.name)

class CLI(BaseSettings):
    model_config = ConfigDict(cli_kebab_case=True, cli_implicit_flags=True)
    sub: CliSubCommand[SubCmd]

def main() -> None:
    try:
        model = CliApp.run(CLI)
        CliApp.run_subcommand(model)
    except SystemExit:
        raise
    except Exception:
        traceback.print_exc()
        raise SystemExit(1) from None
```

## Error Handling

- **Parse errors**: `cli_exit_on_error=True` (default) → `sys.exit(2)`. Set `False` → raises `SettingsError`.
- **Runtime errors** (in `cli_cmd`): propagate through `run_subcommand()`. Do NOT propagate through `CliApp.run()` alone.

## Source Priority (highest → lowest)

1. CLI args  2. Init kwargs  3. Env vars  4. Dotenv  5. Secrets dir  6. Defaults

## Key API

| Method | Purpose |
|--------|---------|
| `CliApp.run(model_cls, cli_args=None, ...)` | Parse args, create model, run root `cli_cmd` |
| `CliApp.run_subcommand(model, ...)` | Find and run active subcommand's `cli_cmd` |
| `get_subcommand(model, is_required=True)` | Get active subcommand model instance |

## Subcommand Rules

- `CliSubCommand` fields must NOT have defaults
- `CliPositionalArg` fields are always required
- Both are always case-sensitive
- Only one subparser group per model
- `BaseModel` defaults differ from `BaseSettings` defaults for CLI options
