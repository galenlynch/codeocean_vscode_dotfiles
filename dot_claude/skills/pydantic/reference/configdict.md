# Pydantic v2 ConfigDict — Complete Reference

## Validation Behavior

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `strict` | `bool` | `False` | Disable type coercion globally |
| `extra` | `str` | `"ignore"` | `"forbid"`, `"allow"`, or `"ignore"` extra fields |
| `frozen` | `bool` | `False` | Faux-immutable instances + hashable |
| `validate_assignment` | `bool` | `False` | Re-validate on attribute assignment |
| `validate_default` | `bool` | `False` | Validate default field values |
| `validate_return` | `bool` | `False` | Validate return values from call validators |
| `revalidate_instances` | `str` | `"never"` | `"always"`, `"subclass-instances"`, or `"never"` |
| `arbitrary_types_allowed` | `bool` | `False` | Allow non-pydantic types in fields |
| `from_attributes` | `bool` | `False` | Enable ORM mode (validate from object attributes) |
| `regex_engine` | `str` | `"rust-regex"` | `"python-re"` for Python regex |

## String Processing

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `str_to_lower` | `bool` | `False` | Lowercase all strings |
| `str_to_upper` | `bool` | `False` | Uppercase all strings |
| `str_strip_whitespace` | `bool` | `False` | Strip whitespace |
| `str_min_length` | `int \| None` | `None` | Min string length |
| `str_max_length` | `int \| None` | `None` | Max string length |

## Aliases

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `validate_by_alias` | `bool` | `True` | Allow population by alias |
| `validate_by_name` | `bool` | `False` | Allow population by field name when alias set |
| `alias_generator` | `Callable \| None` | `None` | Auto-generate aliases from field names |
| `serialize_by_alias` | `bool` | `False` | Default to alias in serialization |
| `loc_by_alias` | `bool` | `True` | Use alias in error locations |

## Serialization & JSON

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `ser_json_timedelta` | `str` | `"iso8601"` | `"float"` for seconds |
| `ser_json_temporal` | `str` | `"iso8601"` | `"seconds"` or `"milliseconds"` |
| `ser_json_bytes` | `str` | `"utf8"` | `"base64"` or `"hex"` |
| `ser_json_inf_nan` | `str` | `"null"` | `"constants"` or `"strings"` |
| `val_json_bytes` | `str` | `"utf8"` | Decoding for JSON bytes |
| `val_temporal_unit` | `str` | `"infer"` | `"seconds"` or `"milliseconds"` |

## JSON Schema

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `title` | `str \| None` | `None` | JSON schema title |
| `json_schema_extra` | `dict \| Callable \| None` | `None` | Extra JSON schema properties |
| `json_schema_serialization_defaults_required` | `bool` | `False` | Default fields required in serialization schema |
| `json_schema_mode_override` | `str \| None` | `None` | `"validation"` or `"serialization"` |

## Type Handling

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `use_enum_values` | `bool` | `False` | Store `.value` not enum instance |
| `coerce_numbers_to_str` | `bool` | `False` | Auto-convert numbers to strings |
| `allow_inf_nan` | `bool` | `True` | Allow infinity/NaN in floats |

## Error & Debug

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `hide_input_in_errors` | `bool` | `False` | Redact input values in errors |
| `validation_error_cause` | `bool` | `False` | Show Python exceptions as causes |
| `use_attribute_docstrings` | `bool` | `False` | Use attribute docstrings for field descriptions |

## Advanced

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `protected_namespaces` | `tuple` | `("model_validate", "model_dump")` | Prevent field name conflicts |
| `defer_build` | `bool` | `False` | Lazy validator/serializer construction |
| `cache_strings` | `bool \| str` | `True` | `"all"`, `"keys"`, `"none"` |
| `ignored_types` | `tuple` | `()` | Types allowed as class attrs without annotations |

## Inheritance

Config merges with parent — child settings override parent, unset options inherit.
Each model has its own config boundary (not propagated to nested models).
