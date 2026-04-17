# Git
- Stage only files that were actually changed — never `git add -A` or `git add .`
- Use conventional commits style (e.g. `feat:`, `fix:`, `chore:`)

# Python
- Always use `uv` — never bare `pip` or `python`. Use `uv add` to add dependencies, `uv sync` to set up environments.
- Never install into the global Python environment.
- The following tools are installed globally and should be invoked directly (not via `uv run`): `ruff`, `mypy`, `codespell`, `interrogate`, `jupytext`.
- Use `uv run` only for commands that need the project venv (e.g. `uv run pytest`).
