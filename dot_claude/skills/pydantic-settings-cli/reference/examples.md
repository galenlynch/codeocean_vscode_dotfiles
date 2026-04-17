# Pydantic Settings CLI — Worked Examples

## Example 1: Multi-Subcommand CLI Tool

```python
from __future__ import annotations
from pathlib import Path
from typing import Any

from pydantic import BaseModel, ConfigDict, Field
from pydantic_settings import BaseSettings, CliApp, CliSubCommand

class BuildCmd(BaseSettings):
    """Build artifacts from source."""
    input: str = Field(description="Input file or directory")
    output: str = Field(description="Output path")
    verbose: bool = Field(default=False, description="Verbose logging")
    workers: int = Field(default=4, description="Parallel workers")

    def cli_cmd(self) -> None:
        print(f"Building {self.input} -> {self.output} (workers={self.workers})")

class DeployCmd(BaseSettings):
    """Deploy artifacts to remote."""
    target: str = Field(description="Deploy target (e.g. s3://bucket/path)")
    dry_run: bool = Field(default=False, description="Simulate without deploying")

    def cli_cmd(self) -> None:
        prefix = "[DRY RUN] " if self.dry_run else ""
        print(f"{prefix}Deploying to {self.target}")

class InfoCmd(BaseSettings):
    """Show project info."""
    path: str = Field(default=".", description="Project path")

    def cli_cmd(self) -> None:
        print(f"Project at {self.path}")

class CLI(BaseSettings):
    """my-tool: Build and deploy artifacts."""
    model_config = ConfigDict(cli_kebab_case=True)
    build: CliSubCommand[BuildCmd]
    deploy: CliSubCommand[DeployCmd]
    info: CliSubCommand[InfoCmd]

def main() -> None:
    try:
        model = CliApp.run(CLI)
        CliApp.run_subcommand(model)
    except SystemExit:
        raise
    except Exception:
        import traceback
        traceback.print_exc()
        raise SystemExit(1) from None
```

Usage:
```bash
my-tool build --input src/ --output dist/ --verbose --workers 8
my-tool deploy --target s3://my-bucket/releases --dry-run
my-tool info
my-tool --help
my-tool build --help
```

## Example 2: CLI with Rich Progress and Error Handling

```python
from rich.console import Console
from rich.progress import Progress

console = Console()

class ProcessCmd(BaseSettings):
    input: str = Field(description="Input file")
    output: str = Field(description="Output file")

    def cli_cmd(self) -> None:
        with Progress(console=console) as progress:
            task = progress.add_task("Processing...", total=100)
            for i in range(100):
                do_work(i)
                progress.update(task, advance=1)
        console.print("[green]Done.[/green]")

def main() -> None:
    try:
        model = CliApp.run(CLI)
        CliApp.run_subcommand(model)
    except SystemExit:
        raise
    except Exception:
        console.print_exception()
        raise SystemExit(1) from None
```

## Example 3: CLI with Env Var Fallbacks

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="MYAPP_",
        cli_parse_args=True,
        cli_kebab_case=True,
    )
    api_key: str = Field(description="API key (or set MYAPP_API_KEY)")
    endpoint: str = Field(default="https://api.example.com", description="API endpoint")
    timeout: int = Field(default=30, description="Request timeout in seconds")

    def cli_cmd(self) -> None:
        print(f"Connecting to {self.endpoint} with timeout={self.timeout}")
```

Priority: `--api-key` CLI flag > `MYAPP_API_KEY` env var > default.

## Example 4: Config Objects Pattern (Reduce Parameter Sprawl)

When a CLI tool has many shared parameters across subcommands, use frozen dataclasses as config bundles:

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class BuildConfig:
    workers: int = 4
    cache_dir: str = ".cache"
    verbose: bool = False

class BuildCmd(BaseSettings):
    input: str = Field(description="Input path")
    config: BuildConfig = BuildConfig()  # grouped params

    def cli_cmd(self) -> None:
        cfg = self.config
        ...
```

## Example 5: Custom Type with CLI Choices

```python
from enum import Enum
from typing import Literal

class OutputFormat(str, Enum):
    json = "json"
    csv = "csv"
    parquet = "parquet"

class ExportCmd(BaseSettings):
    format: OutputFormat = Field(default=OutputFormat.json, description="Output format")
    # CLI shows: --format {json,csv,parquet}

    compression: Literal["none", "gzip", "zstd"] = Field(default="none")
    # CLI shows: --compression {none,gzip,zstd}
```
