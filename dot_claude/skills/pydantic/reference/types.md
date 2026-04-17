# Pydantic v2 Types & Strict Mode

## Type Coercion Rules (lax mode, default)

| Target type | Accepts | Coercion |
|-------------|---------|----------|
| `int` | `str`, `float`, `bool` | `"42"` → `42`, `3.0` → `3`, `True` → `1` |
| `float` | `str`, `int`, `bool` | `"3.14"` → `3.14`, `42` → `42.0` |
| `str` | most types | via `str()` |
| `bool` | `int`, `str` | `1`/`0`, `"true"`/`"false"` |
| `datetime` | `str`, `int`, `float` | ISO 8601 strings, Unix timestamps |
| `date` | `str` | ISO 8601 date strings |
| `Decimal` | `str`, `int`, `float` | string/numeric conversion |

## Strict Mode

Disables coercion — input must match the exact declared type.

```python
# Per-field
class Model(BaseModel):
    value: int = Field(strict=True)

# Per-model
class Model(BaseModel):
    model_config = ConfigDict(strict=True)

# Per-call
Model.model_validate(data, strict=True)

# Convenience aliases
from pydantic import StrictStr, StrictInt, StrictFloat, StrictBool, StrictBytes
```

Exception: strict mode is **looser for JSON** — `datetime` accepts ISO strings even in strict mode.

## Constrained Types

```python
from pydantic import Field
from typing import Annotated

# Via Field
class Model(BaseModel):
    age: int = Field(ge=0, le=150)
    name: str = Field(min_length=1, max_length=100, pattern=r"^[a-zA-Z]+$")
    score: float = Field(gt=0, lt=1.0)

# Reusable via Annotated
PositiveInt = Annotated[int, Field(gt=0)]
ShortStr = Annotated[str, Field(max_length=50)]
```

## Special Pydantic Types

```python
from pydantic import SecretStr, SecretBytes, FilePath, DirectoryPath, NewPath
from pydantic import HttpUrl, AnyUrl, EmailStr  # EmailStr needs pydantic[email]
from pydantic import PositiveInt, NegativeInt, NonNegativeInt, NonPositiveInt
from pydantic import PositiveFloat, NegativeFloat
from pydantic import conint, confloat, constr  # constrained type constructors
```

## Enum and Literal

```python
from enum import Enum, IntEnum
from typing import Literal

class Color(str, Enum):
    red = "red"
    green = "green"

class Model(BaseModel):
    color: Color                    # accepts "red" or Color.red
    mode: Literal["fast", "slow"]   # only these exact strings
```

With `use_enum_values=True` in ConfigDict, stores `.value` not the enum instance.

## Union Types & Discriminated Unions

```python
# Simple union — tried left to right
class Model(BaseModel):
    value: int | str

# Discriminated union — efficient, no ambiguity
from typing import Annotated, Union
from pydantic import Discriminator, Tag

class Cat(BaseModel):
    pet_type: Literal["cat"]
    meow: str

class Dog(BaseModel):
    pet_type: Literal["dog"]
    bark: str

class Model(BaseModel):
    pet: Cat | Dog = Field(discriminator="pet_type")
```

## Optional vs Required

```python
class Model(BaseModel):
    required: str              # must be provided, cannot be None
    optional: str | None       # must be provided, CAN be None
    defaulted: str = "hello"   # not required, has default
    opt_default: str | None = None  # not required, defaults to None
```

## Custom Types

Implement `__get_pydantic_core_schema__` for full control:

```python
from pydantic import GetCoreSchemaHandler
from pydantic_core import CoreSchema, core_schema

class MyType:
    def __init__(self, value: str) -> None:
        self.value = value

    @classmethod
    def __get_pydantic_core_schema__(cls, source_type, handler: GetCoreSchemaHandler) -> CoreSchema:
        return core_schema.no_info_plain_validator_function(
            lambda v: cls(v) if isinstance(v, str) else v,
        )
```

## Dataclasses

```python
from pydantic.dataclasses import dataclass

@dataclass
class User:
    id: int
    name: str = "default"
```

- Adds validation to stdlib dataclass syntax
- NOT a full replacement for BaseModel (missing some features)
- Use `TypeAdapter(MyDataclass)` for validation/serialization methods
- `extra` config behaves differently than BaseModel
