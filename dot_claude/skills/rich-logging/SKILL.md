---
name: rich-logging
description: Integrate rich's Progress/Live displays with Python logging without clobbering each other. Use when writing or reviewing code that imports both `rich.progress`/`rich.live` AND the standard `logging` module. The specific trap: `logger.exception(...)` called inside a `with Progress(...):` block has its traceback tail overwritten by rich's Live-display tick, silently clipping the actual exception line and making failures un-debuggable.
---

# Rich + Logging

Rich's `Progress` and `Live` displays repaint their live area 2–10× per second. If Python's standard logging emits a record (especially a multi-line `logger.exception(...)` traceback) while Live is active **and** logging isn't routed through rich, the next rich tick paints over the tail of the log output. Symptom: exception tracebacks that end at the last Python frame with no `ErrorType: message` line — the diagnostic is silently gone.

This has bitten real code multiple times. Not theoretical.

## The fix

Route `logging` through `rich.logging.RichHandler`, sharing the **same** `Console` instance that `Progress`/`Live` uses. Rich then serializes log output with the live area (pauses Live, prints log record above, resumes Live).

```python
from rich.console import Console
from rich.logging import RichHandler
from rich.progress import Progress
import logging, sys

# Module-level Console, shared by logging AND Progress/Live:
_console = Console(force_terminal=sys.stdout.isatty(), stderr=False)

def _init_logging() -> None:
    logging.basicConfig(
        level=logging.INFO,
        format="%(name)s: %(message)s",
        datefmt="[%X]",
        handlers=[RichHandler(console=_console, rich_tracebacks=True, show_path=False)],
    )

def main():
    _init_logging()
    log = logging.getLogger("mytool")

    with Progress(console=_console) as progress:   # same _console
        task = progress.add_task("working", total=100)
        try:
            do_work(progress, task)
        except Exception:
            log.exception("work failed")            # traceback renders cleanly
```

Key points:

- **One `Console` instance, shared.** If `Progress` constructs its own Console separately from the logging handler, they don't know about each other and the bug returns.
- **`RichHandler(rich_tracebacks=True)`** formats exceptions with syntax highlighting and local-variable context — free upgrade over stdlib traceback rendering.
- **`show_path=False`** keeps log lines compact; turn on for debug builds.

## When to reach for this skill

Any time code does both:

1. `from rich.progress import Progress` / `from rich.live import Live`
2. `import logging` with `getLogger(...)` / `logger.exception` / `logger.error`

Before writing or reviewing such code, check that logging is routed through `RichHandler` with the shared Console. If not, flag it — the bug is latent until the first exception inside the Progress/Live block, at which point the actual error is invisible.

## What NOT to do

- **Don't** call `progress.stop()` before logging as the fix. That works but loses the live display for the rest of the run.
- **Don't** catch and manually `print(traceback.format_exc())` to work around clobbering — RichHandler is less code and handles all log calls uniformly.
- **Don't** use two separate `Console` instances "one for logs, one for progress" — they'll each tick on top of each other.
