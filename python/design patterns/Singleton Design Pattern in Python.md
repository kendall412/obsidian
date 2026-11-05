
### 1. What is the Singleton Design Pattern?

The Singleton Design Pattern is a creational design pattern that ensures:
1. Only one instance of a class is created.
2. All parts of a program share that same instance.
3. The class provides a way to access that instance.

In simple terms, no matter how many times you try to create an object from a Singleton class, you always receive the same object.

#### Normal class vs. Singleton class

![[normal vs singleton.png]]


### 2. Simple Python example

Here is a basic Singleton implementation using Python's `__new__()` method.

```python
class Singleton:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)

        return cls._instance


# Create objects
obj1 = Singleton()
obj2 = Singleton()
obj3 = Singleton()

# Check whether they are the same object
print(obj1 is obj2)
print(obj2 is obj3)

```

Output:
```
True
True
```

All three variables reference the same object in memory.

### 3. Explanation line by line

Step 1: Declare a class variable

```python
_instance = None
```

- `_instance` stores the single object
- Initially, no object exists, so its value is `None`
- Because it is a class variable, it is shared at the class level


Step 2: Override `__new__()`

```python
def __new__(cls):
```

- `__new__()` is responsible for creating an object
- `cls` refers to the class being instantiated
- Python calls `__new__()` before `__init__()`
    

Step 3: Check whether an instance already exists

```python
if cls._instance is None:
    cls._instance = super().__new__(cls)
```

- If no instance exists, create one using `super().__new__(cls)`
- Store the new object in `_instance`
- If an instance already exists, skip object creation
    

Step 4: Return the instance

```python
return cls._instance
```

This always returns the previously created instance, or the newly created instance on the first call.

### 4. Practical example: Logger

Suppose you are developing an NVMe validation automation framework. Multiple test modules need to write messages to the same logger.

Instead of creating a separate logger object for each module, you can use a Singleton.

```python
class Logger:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)

        return cls._instance

    def log(self, message):
        print(f"[LOG] {message}")


# Test module 1
logger1 = Logger()
logger1.log("Starting NVMe test")

# Test module 2
logger2 = Logger()
logger2.log("Executing Write command")

# Verify same instance
print(logger1 is logger2)

```

Output:
```
[LOG] Starting NVMe test
[LOG] Executing Write command
True
```

Both test modules share the same logger object.

Note that this example prints messages rather than writing them to a log file. A production logger would typically manage a file handler, formatting, and concurrency.

### 5. When should you use Singleton?

Singleton is useful when an application needs one shared instance of a service or resource.

|Use case|Why Singleton might help|
|---|---|
|Logger|Centralized logging configuration|
|Configuration manager|Shared application settings|
|Device manager|Centralized access to an NVMe device|
|Cache manager|Shared in-memory cache|
|Resource manager|Coordinated access to shared resources|

#### Important limitations

- Singleton introduces shared global state, which can make unit testing harder.
- A basic Singleton is not automatically thread-safe.
- Python calls `__init__()` on every normal construction attempt, even when `__new__()` returns an existing object.
- Singleton generally guarantees one instance per Python process, not one across multiple processes.

### 6. Singleton vs. Factory Design Pattern

|Feature|Singleton|Factory|
|---|---|---|
|Main purpose|Ensure one shared instance|Encapsulate object creation|
|Number of objects|One instance per Singleton class|Potentially many|
|Object selection|Returns existing instance|Can select different classes|
|Example|One shared logger|Create SATA, NVMe, or SAS device objects|

Key takeaway: A Factory determines which object to create, while a Singleton determines whether a new instance should be created at all.

In an NVMe validation framework, you might use a Factory to select the appropriate controller implementation and a Singleton to manage one shared logging service.

