---
name: pydantic
description: Pydantic v2 patterns, validators, serialization, ConfigDict, types, and gotchas. Use when writing or debugging pydantic models, validators, serializers, or configuration.
---

# Pydantic v2

Reference for `pydantic>=2.0`. This skill covers models, validators, serialization, config, and common pitfalls.

## When to consult reference files

- **Validators** (field_validator, model_validator, Annotated validators, modes, ordering): see `reference/validators.md`
- **Serialization** (model_dump, field_serializer, model_serializer, computed_field, aliases): see `reference/serialization.md`
- **ConfigDict** (all options with defaults): see `reference/configdict.md`
- **Types and strict mode** (coercion rules, strict types, custom types): see `reference/types.md`
- **Gotchas** (v1→v2 migration, common mistakes): see `reference/gotchas.md`

## Essential patterns (quick ref)

```python
from pydantic import BaseModel, ConfigDict, Field, field_validator, model_validator
from typing import Annotated
from typing_extensions import Self

class Model(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")
    name: str = Field(min_length=1, description="shown in JSON schema")

    @field_validator("name", mode="after")
    @classmethod
    def must_be_alpha(cls, v: str) -> str:
        if not v.isalpha(): raise ValueError("must be alphabetic")
        return v

    @model_validator(mode="after")
    def check_invariants(self) -> Self:
        ...
        return self

# Serialization
d = model.model_dump(exclude_unset=True, mode="json")
j = model.model_dump_json(indent=2)

# Reusable constrained types
PositiveInt = Annotated[int, Field(gt=0)]
```
