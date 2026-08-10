In Python 3, a **dataclass** is a class designed mainly to store data, with common methods automatically generated for you.

It comes from the `dataclasses` module, introduced in Python 3.7.

Example:

```python
from dataclasses import dataclass

@dataclass
class Person:
    name: str
    age: int
```

This automatically creates an `__init__` method, so you can do:

```python
p = Person("Alice", 30)

print(p.name)  # Alice
print(p.age)   # 30
print(p)       # Person(name='Alice', age=30)
```

Without `@dataclass`, you would normally write something like:

```python
class Person:
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age
```

A dataclass saves you from writing repetitive boilerplate code.

Common features include:

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int
```

Automatically gives you:

- `__init__`
- `__repr__`
- `__eq__`

So:

```python
p1 = Point(1, 2)
p2 = Point(1, 2)

print(p1)       # Point(x=1, y=2)
print(p1 == p2) # True
```

You can also provide default values:

```python
@dataclass
class Config:
    host: str = "localhost"
    port: int = 8080
```

Usage:

```python
c = Config()
print(c.host)  # localhost
print(c.port)  # 8080
```

For mutable defaults like lists, use `field(default_factory=...)`:

```python
from dataclasses import dataclass, field

@dataclass
class Team:
    name: str
    members: list[str] = field(default_factory=list)
```

This avoids sharing the same list between all instances.

You can also make a dataclass immutable:

```python
@dataclass(frozen=True)
class Coordinate:
    x: int
    y: int
```

Then this will fail:

```python
c = Coordinate(1, 2)
c.x = 10  # Error
```

In short, `@dataclass` is a convenient way to define simple classes that mostly hold data.