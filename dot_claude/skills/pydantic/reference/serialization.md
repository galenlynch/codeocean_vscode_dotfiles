# Pydantic v2 Serialization

## model_dump() Options

```python
model.model_dump(
    include={"name", "id"},          # only these fields
    exclude={"password"},             # skip these fields
    exclude_unset=True,               # omit fields not explicitly set
    exclude_defaults=True,            # omit fields matching their default
    exclude_none=True,                # omit None-valued fields
    by_alias=True,                    # use serialization_alias
    mode="json",                      # JSON-compatible types (tuples→lists, datetime→str)
    round_trip=True,                  # ensure model_validate(model_dump()) roundtrips
)
```

Nested exclusion:
```python
model.model_dump(exclude={"address": {"zip_code"}})
model.model_dump(include={"name": True, "address": {"city"}})
# All items in a list: exclude={"items": {"__all__": {"internal_id"}}}
```

## model_dump_json()

Same options as `model_dump()` plus `indent` for formatting:
```python
model.model_dump_json(indent=2, exclude_none=True)
```

## Field-Level Exclusion

```python
class Model(BaseModel):
    public: str
    secret: str = Field(exclude=True)           # always excluded
    maybe: int = Field(exclude_if=lambda v: v == 0)  # conditional
```

## Field Serializers

### Plain (replaces default serialization)
```python
from pydantic import field_serializer

class Model(BaseModel):
    value: float

    @field_serializer("value", mode="plain")
    def round_it(self, v: float) -> float:
        return round(v, 2)
```

### Wrap (wraps default serialization)
```python
from pydantic import field_serializer, SerializerFunctionWrapHandler

class Model(BaseModel):
    value: int

    @field_serializer("value", mode="wrap")
    def add_one(self, v: int, handler: SerializerFunctionWrapHandler) -> int:
        return handler(v) + 1
```

### Annotated (reusable)
```python
from pydantic import PlainSerializer, WrapSerializer

RoundedFloat = Annotated[float, PlainSerializer(lambda v: round(v, 2))]
```

### With context
```python
@field_serializer("text", mode="plain")
@classmethod
def filter(cls, v: str, info: FieldSerializationInfo) -> str:
    ctx = info.context or {}
    # use ctx to customize serialization
    return v

model.model_dump(context={"format": "compact"})
```

## Model Serializers

### Plain (complete control)
```python
from pydantic import model_serializer

class Model(BaseModel):
    name: str
    value: int

    @model_serializer(mode="plain")
    def ser(self) -> str:
        return f"{self.name}={self.value}"
```

### Wrap (modify default output)
```python
@model_serializer(mode="wrap")
def ser(self, handler: SerializerFunctionWrapHandler) -> dict:
    d = handler(self)
    d["_type"] = type(self).__name__
    return d
```

## Computed Fields

```python
from pydantic import computed_field

class Rect(BaseModel):
    w: float
    h: float

    @computed_field
    @property
    def area(self) -> float:
        return self.w * self.h
```

Included in `model_dump()`, JSON schema. Read-only.

## Serialization Aliases

```python
class Model(BaseModel):
    user_id: int = Field(serialization_alias="id")

model.model_dump()              # {"user_id": 1}
model.model_dump(by_alias=True) # {"id": 1}
```

Separate from validation aliases — a field can have both.

## Duck-Typed Serialization

```python
from pydantic import SerializeAsAny

class Parent(BaseModel):
    x: int

class Child(Parent):
    y: int

class Container(BaseModel):
    item: SerializeAsAny[Parent]  # serializes Child fields too

# Or at call site:
container.model_dump(serialize_as_any=True)
```

## model_copy()

```python
new = model.model_copy(update={"name": "new_name"}, deep=True)
```
