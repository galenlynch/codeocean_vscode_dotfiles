# Pydantic Settings ConfigDict Options — Complete Reference

## CLI-Specific Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `cli_parse_args` | `bool \| list[str]` | `False` | Enable CLI parsing. `True` reads `sys.argv[1:]` |
| `cli_exit_on_error` | `bool` | `True` | Exit on parse error. `False` raises `SettingsError` |
| `cli_prog_name` | `str \| None` | `sys.argv[0]` | Program name in help text |
| `cli_kebab_case` | `bool \| "all"` | `False` | Convert field names to kebab-case in CLI |
| `cli_implicit_flags` | `bool` | `False` | Boolean flags as `--flag`/`--no-flag` |
| `cli_avoid_json` | `bool` | `False` | Expand complex types to nested args instead of JSON |
| `cli_enforce_required` | `bool` | `False` | Require CLI-supplied values even if env provides them |
| `cli_hide_none_type` | `bool` | `False` | Hide `None` type from help text |
| `cli_ignore_unknown_args` | `bool` | `False` | Ignore unrecognized arguments |
| `cli_use_class_docs_for_groups` | `bool` | `False` | Use model docstrings for group help text |
| `cli_flag_prefix_char` | `str` | `'-'` | Prefix character for flags |
| `cli_parse_none_str` | `str` | `'null'` or `'None'` | String value parsed as `None` |
| `cli_shortcuts` | `dict` | `{}` | Map field names to short aliases |
| `cli_prefix` | `str \| None` | `None` | Command argument prefix |

## Environment Variable Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `env_prefix` | `str` | `''` | Prefix for env var names |
| `env_nested_delimiter` | `str \| None` | `None` | Delimiter for nested env vars (e.g. `'__'`) |
| `env_nested_max_split` | `int` | unlimited | Max nesting depth for delimiter splitting |
| `env_file` | `str \| Path \| tuple` | `None` | Dotenv file path(s) |
| `env_file_encoding` | `str \| None` | `None` | Encoding for dotenv files |
| `env_ignore_empty` | `bool` | `False` | Ignore empty string env vars |
| `env_parse_none_str` | `str \| None` | `None` | String value parsed as `None` |
| `env_parse_enums` | `bool \| None` | `None` | Parse enum values from env vars |
| `case_sensitive` | `bool` | `False` | Case-sensitive env var matching |

## Secrets Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `secrets_dir` | `str \| Path \| None` | `None` | Directory containing secret files |

## General BaseSettings Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `nested_model_default_partial_update` | `bool` | `False` | Allow partial updates to nested model defaults |

## Note on BaseModel vs BaseSettings defaults

When using `CliApp.run()` with a `BaseModel` (not `BaseSettings`), these defaults change:
- `nested_model_default_partial_update=True`
- `case_sensitive=True`
- `cli_hide_none_type=True`
- `cli_avoid_json=True`
- `cli_enforce_required=True`
- `cli_implicit_flags=True`
- `cli_kebab_case=True`
