# Pydantic v2 Gotchas & Migration

## Common Mistakes

### 1. Defaults aren't validated
```python
class Model(BaseModel):
    value: int = "not_an_int"  # Silently accepted!
# Fix: model_config = ConfigDict(validate_default=True)
```

### 2. Field access order in validators
```python
class Model(BaseModel):
    a: int
    b: int

    @field_validator("a")
    @classmethod
    def check_a(cls, v, info):
        info.data["b"]  # KeyError! b not validated yet
```
Fields validate in definition order. Validator for `a` cannot access `b`.

### 3. model_dump() vs dict()
```python
dict(model)           # shallow — nested models stay as model instances
model.model_dump()    # recursive — nested models become dicts
```

### 4. Frozen != truly immutable
```python
class Model(BaseModel):
    model_config = ConfigDict(frozen=True)
    items: list[int]

m = Model(items=[1, 2])
m.items.append(3)  # Works! The list itself is mutable
# m.items = [4]    # This raises ValidationError
```

### 5. Name collisions with builtins
```python
class Model(BaseModel):
    int: int | None = None    # Shadows int type!
    list: list[str] = []       # Shadows list type!
```

### 6. Mutable default values
```python
# WRONG:
class Model(BaseModel):
    items: list[int] = []  # Shared across instances... but pydantic handles this!
# Actually OK in pydantic — it copies defaults. But prefer:
class Model(BaseModel):
    items: list[int] = Field(default_factory=list)
```

### 7. Union type ordering
```python
class Model(BaseModel):
    value: int | str  # "42" → int(42), not str("42")
    value: str | int  # "42" → str("42")
```
Unions try left-to-right. Order matters.

### 8. exclude_unset vs exclude_defaults
```python
class Model(BaseModel):
    x: int = 0

m = Model(x=0)
m.model_dump(exclude_unset=True)    # {"x": 0} — x WAS set (explicitly)
m.model_dump(exclude_defaults=True) # {} — x equals default
```

## v1 → v2 Migration

| v1 | v2 |
|----|----|
| `class Config:` | `model_config = ConfigDict(...)` |
| `@validator("field")` | `@field_validator("field")` |
| `@root_validator` | `@model_validator` |
| `.parse_obj(data)` | `.model_validate(data)` |
| `.parse_raw(json_str)` | `.model_validate_json(json_str)` |
| `.dict()` | `.model_dump()` |
| `.json()` | `.model_dump_json()` |
| `.copy()` | `.model_copy()` |
| `.schema()` | `.model_json_schema()` |
| `.update_forward_refs()` | `.model_rebuild()` |
| `allow_mutation = False` | `frozen = True` |
| `orm_mode = True` | `from_attributes = True` |
| `schema_extra` | `json_schema_extra` |
| `validate_all = True` | `validate_default = True` |
| `allow_population_by_field_name` | `validate_by_name` |

## Performance Tips

- `defer_build=True` — delay schema construction until first use (faster imports)
- `cache_strings=True` — reuse string objects (default, reduces memory)
- `model_validate_json()` is faster than `model_validate(json.loads(...))` — avoids intermediate Python dict
- Use discriminated unions over plain unions for complex type hierarchies
