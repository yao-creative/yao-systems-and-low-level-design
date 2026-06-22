Python has several mechanisms that act as mixins:

* **Multiple inheritance** (most common)
* **Abstract base classes (ABCs)** for constrained mixins
* **Protocols** (typing) for structural constraints
* `dataclass` inheritance for state augmentation

The key pattern is:

```python
class CapabilityMixin:
    def capability(self):
        ...
```

and then:

```python
class DomainObject(CapabilityMixin):
    ...
```

The mixin is **not meant to stand alone**; it assumes a host type.

---

# 1. LoggingMixin (behavior extension)

Adds diagnostic capability.

```python
class LoggingMixin:
    def log(self, message):
        print(f"[{self.__class__.__name__}] {message}")


class User(LoggingMixin):
    def __init__(self, name):
        self.name = name


user = User("Alice")

user.log("created")
```

Output:

```
[User] created
```

Formal view:

[
M(A)=A \oplus {log}
]

---

# 2. SerializableMixin (cross-cutting capability)

Assumes the object has fields in `__dict__`.

```python
import json


class JsonMixin:
    def to_json(self):
        return json.dumps(self.__dict__)


class User(JsonMixin):
    def __init__(self, name, age):
        self.name = name
        self.age = age


print(User("Alice", 20).to_json())
```

Output:

```json
{"name": "Alice", "age": 20}
```

The mixin requires:

[
P(A)=\text{has attribute dictionary}
]

---

# 3. TimestampMixin (state extension)

A mixin can add state.

```python
from datetime import datetime


class TimestampMixin:
    def __init__(self):
        self.created_at = datetime.now()


class User(TimestampMixin):
    def __init__(self, name):
        super().__init__()
        self.name = name


u = User("Alice")

print(u.created_at)
```

This adds:

```
created_at : datetime
```

---

# 4. EqualityMixin (derived behavior)

The class only provides the primitive representation.

```python
class EqualityMixin:
    def __eq__(self, other):
        return self.key() == other.key()


class User(EqualityMixin):
    def __init__(self, user_id):
        self.user_id = user_id

    def key(self):
        return self.user_id


a = User(1)
b = User(1)

print(a == b)
```

Output:

```
True
```

Here:

Required:

```python
key()
```

Provided:

```python
__eq__()
```

Formalized as:

[
P(A)={\text{key}:A\to K}
]

[
M(A)=A\oplus{==}
]

---

# 5. ComparableMixin (derive a whole interface)

This is a classic mixin pattern.

Provide one primitive:

```python
compare(other)
```

Get many operations.

```python
class ComparableMixin:
    def __lt__(self, other):
        return self.compare(other) < 0

    def __le__(self, other):
        return self.compare(other) <= 0

    def __gt__(self, other):
        return self.compare(other) > 0

    def __ge__(self, other):
        return self.compare(other) >= 0


class Money(ComparableMixin):
    def __init__(self, amount):
        self.amount = amount

    def compare(self, other):
        return self.amount - other.amount


print(Money(5) < Money(10))
```

Output:

```
True
```

This is exactly the pattern:

[
\text{primitive operations} \Rightarrow \text{derived algebra}
]

---

# 6. SoftDeleteMixin (state + behavior)

Very common in ORM systems.

```python
class SoftDeleteMixin:
    def __init__(self):
        self.deleted = False

    def delete(self):
        self.deleted = True

    def restore(self):
        self.deleted = False


class User(SoftDeleteMixin):
    def __init__(self, name):
        super().__init__()
        self.name = name


u = User("Alice")

u.delete()

print(u.deleted)
```

Output:

```
True
```

---

# 7. Combining multiple mixins (the real power)

A class can compose many orthogonal capabilities.

```python
class User(
    LoggingMixin,
    JsonMixin,
    SoftDeleteMixin,
):
    def __init__(self, name):
        SoftDeleteMixin.__init__(self)
        self.name = name


u = User("Alice")

u.log("created")
print(u.to_json())

u.delete()
print(u.deleted)
```

Result:

```
[User] created
{"name":"Alice","deleted":false}
True
```

Conceptually:

[
User'
=====

SoftDelete(
Json(
Logging(User)))
]

---

# 8. A more robust Python style: cooperative inheritance

Large frameworks usually avoid explicitly naming the parent constructor.

They use `super()` and keyword arguments.

```python
class TimestampMixin:
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.created_at = datetime.now()


class SoftDeleteMixin:
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.deleted = False


class User(TimestampMixin, SoftDeleteMixin):
    def __init__(self, name):
        super().__init__()
        self.name = name
```

Python's Method Resolution Order (MRO) handles:

```
User
 ↓
TimestampMixin
 ↓
SoftDeleteMixin
 ↓
object
```

Each mixin contributes its piece of initialization.

---

# The most common industrial mixin patterns

| Pattern         | Adds                 | Examples       |
| --------------- | -------------------- | -------------- |
| LoggingMixin    | diagnostics          | services, jobs |
| JsonMixin       | serialization        | APIs, caching  |
| TimestampMixin  | created/updated time | ORMs           |
| SoftDeleteMixin | deletion lifecycle   | databases      |
| EqualityMixin   | `==`, `hash`         | value objects  |
| ComparableMixin | `<`, `>`, sorting    | domain objects |
| ValidationMixin | `validate()`         | DTOs, configs  |
| ObservableMixin | events               | GUIs, agents   |
| ThreadSafeMixin | locks                | caches, queues |

---

The deepest insight is that a Python mixin is essentially **a reusable implementation of a partial algebra**:

[
\boxed{
\text{Host provides primitive operations}
\quad\Rightarrow\quad
\text{Mixin derives a larger interface}
}
]

This is why mixins are extremely common in ORMs, web frameworks, distributed systems, and agent frameworks: they let you compose independent capabilities without forcing a single inheritance hierarchy.
