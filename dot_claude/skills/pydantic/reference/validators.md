# Pydantic v2 Validators

## Field Validators

Four modes via `@field_validator("field_name", mode=...)`:

### `mode="after"` (default) — runs after type coercion
```python
@field_validator("name", mode="after")
@classmethod
def must_be_alpha(cls, v: str) -> str:
    if not v.isalpha():
        raise ValueError("must be alphabetic")
    return v
```

### `mode="before"` — runs on raw input before coercion
```python
@field_validator("numbers", mode="before")
@classmethod
def ensure_list(cls, v: Any) -> Any:
    return [v] if not isinstance(v, list) else v
```

### `mode="wrap"` — wraps other validators, can catch/retry
```python
@field_validator("value", mode="wrap")
@classmethod
def clamp(cls, v: Any, handler: ValidatorFunctionWrapHandler) -> str:
    try:
        return handler(v)
    except ValidationError:
        return handler(str(v)[:100])  # retry with truncated
```

### `mode="plain"` — replaces all other validation
```python
@field_validator("value", mode="plain")
@classmethod
def custom_parse(cls, v: Any) -> int:
    return int(v) * 2
```

### Multiple fields, all fields
```python
@field_validator("f1", "f2", mode="before")
@classmethod
def capitalize(cls, v: str) -> str: return v.capitalize()

@field_validator("*")  # all fields
@classmethod
def check_all(cls, v): return v
```

### Access to previously validated fields
```python
@field_validator("password_repeat", mode="after")
@classmethod
def passwords_match(cls, v: str, info: ValidationInfo) -> str:
    if v != info.data["password"]:
        raise ValueError("passwords don't match")
    return v
```
Note: can only access fields defined EARLIER in the class body.

### Validation context
```python
@field_validator("text", mode="after")
@classmethod
def filter(cls, v: str, info: ValidationInfo) -> str:
    ctx = info.context or {}
    stopwords = ctx.get("stopwords", set())
    return " ".join(w for w in v.split() if w not in stopwords)

# Pass context:
Model.model_validate(data, context={"stopwords": {"the", "a"}})
```

## Model Validators

### `mode="after"` — instance method, all fields already validated
```python
@model_validator(mode="after")
def check_range(self) -> Self:
    if self.start >= self.end:
        raise ValueError("start must be < end")
    return self
```

### `mode="before"` — classmethod, receives raw data dict
```python
@model_validator(mode="before")
@classmethod
def preprocess(cls, data: Any) -> Any:
    if isinstance(data, dict):
        data.pop("internal_field", None)
    return data
```

### `mode="wrap"` — classmethod with handler, can catch and modify
```python
@model_validator(mode="wrap")
@classmethod
def log_failures(cls, data: Any, handler: ModelWrapValidatorHandler[Self]) -> Self:
    try:
        return handler(data)
    except ValidationError:
        logging.error("Validation failed for %s", data)
        raise
```

## Annotated Validators (reusable)

```python
from pydantic import AfterValidator, BeforeValidator, WrapValidator, PlainValidator

PositiveInt = Annotated[int, AfterValidator(lambda v: v if v > 0 else (_ for _ in ()).throw(ValueError("must be positive")))]
# Or cleaner:
def check_positive(v: int) -> int:
    if v <= 0: raise ValueError("must be positive")
    return v
PositiveInt = Annotated[int, AfterValidator(check_positive)]
```

Stacking: Before/Wrap run right-to-left, then After left-to-right.

## Error Types

- `ValueError` — most common, becomes `value_error` type
- `AssertionError` — `assert x > 0, "msg"`
- `PydanticCustomError(type, template, context)` — custom error type and message
- `PydanticUseDefault()` — signal to use field's default value

## Utilities

- `InstanceOf[Cls]` — validate isinstance without coercion
- `SkipValidation[T]` — bypass all validation for a field
