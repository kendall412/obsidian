
### 1. What is the Factory Design Pattern?
The Factory Design Pattern is a creational design pattern that provides a centralized way to create objects without requiring the code requesting those objects to know exactly how they are created.

In simple terms:
Instead of creating objects directly, you ask a factory to create the appropriate object for you.

For example, suppose you have three classes:
- `Dog`
- `Cat`
- `Cow`

Without a factory, you would create objects directly:
```python
dog = Dog()
cat = Cat()
cow = Cow()
```

With a factory, you can create objects based on an input:
```python
animal = AnimalFactory.create_animal("dog")
```

The factory determines which class to instantiate.

### 2. Simple Factory example in Python

```python
# Step 1: Define different animal classes

class Dog:
    def speak(self):
        return "Woof!"


class Cat:
    def speak(self):
        return "Meow!"


class Cow:
    def speak(self):
        return "Moo!"


# Step 2: Create a Factory class

class AnimalFactory:

    @staticmethod
    def create_animal(animal_type):

        if animal_type == "dog":
            return Dog()

        elif animal_type == "cat":
            return Cat()

        elif animal_type == "cow":
            return Cow()

        else:
            raise ValueError("Unknown animal type")


# Step 3: Use the Factory

animal1 = AnimalFactory.create_animal("dog")
animal2 = AnimalFactory.create_animal("cat")
animal3 = AnimalFactory.create_animal("cow")

print(animal1.speak())
print(animal2.speak())
print(animal3.speak())

```

Output:
```
Woof!
Meow!
Moo!
```

### 3. How the Factory works

![[how factory works.png]]

The execution flow is:
1. The client calls `AnimalFactory.create_animal("dog")`
2. The factory receives `"dog"` as an argument
3. The factory evaluates the `if` conditions
4. Since the input is `"dog"`, it executes `return Dog()`
5. The newly created `Dog` object is returned to the client

The client does not need to call `Dog()` directly.

### 4. Why use `@staticmethod`?

In the example:
```python
class AnimalFactory:

    @staticmethod
    def create_animal(animal_type):
        ...
```

`@staticmethod` allows you to call the method directly from the class without creating an instance of `AnimalFactory`.

```
animal = AnimalFactory.create_animal("dog")
```

You do not need to write:
```python
factory = AnimalFactory()
animal = factory.create_animal("dog")
```

A static method is appropriate here because the factory method does not need `self` or `cls`.

### 5. A better Factory implementation using a dictionary
As the number of supported classes increases, writing many `if` and `elif` statements becomes difficult to maintain.

A dictionary provides a cleaner approach.

```python
class Dog:
    def speak(self):
        return "Woof!"


class Cat:
    def speak(self):
        return "Meow!"


class Cow:
    def speak(self):
        return "Moo!"


class AnimalFactory:

    animal_types = {
        "dog": Dog,
        "cat": Cat,
        "cow": Cow
    }

    @classmethod
    def create_animal(cls, animal_type):

        animal_class = cls.animal_types.get(animal_type)

        if animal_class is None:
            raise ValueError("Unknown animal type")

        return animal_class()


animal = AnimalFactory.create_animal("dog")

print(animal.speak())

```

Output:
```
Woof!
```

#### Explanation of the important lines

First:
```python
animal_types = {
    "dog": Dog,
    "cat": Cat,
    "cow": Cow
}
```

This dictionary stores references to classes, not instances.

Notice the difference:
```python
"dog": Dog     # Class reference
"dog": Dog()   # Instance object
```

Second:
```python
animal_class = cls.animal_types.get(animal_type)
```

If `animal_type` is `"dog"`, then `animal_class` refers to the `Dog` class.

Third:
```python
return animal_class()
```

This creates a new instance of the selected class.

It is equivalent to:
```python
return Dog()
```

This implementation uses `@classmethod` because it accesses the class-level registry through `cls`.

### 6. Practical example: NVMe validation framework
Suppose you have an SSD validation framework that supports different types of tests.

You could define three test classes:
```python
class ReadTest:
    def run(self):
        print("Running NVMe Read Test")


class WriteTest:
    def run(self):
        print("Running NVMe Write Test")


class FormatTest:
    def run(self):
        print("Running NVMe Format Test")
```

Instead of creating the test objects throughout your automation framework, you can use a factory.

```python
class ReadTest:
    def run(self):
        print("Running NVMe Read Test")


class WriteTest:
    def run(self):
        print("Running NVMe Write Test")


class FormatTest:
    def run(self):
        print("Running NVMe Format Test")


class TestFactory:

    test_types = {
        "read": ReadTest,
        "write": WriteTest,
        "format": FormatTest
    }

    @classmethod
    def create_test(cls, test_type):

        test_class = cls.test_types.get(test_type)

        if test_class is None:
            raise ValueError("Invalid test type")

        return test_class()


# Test execution
test1 = TestFactory.create_test("read")
test2 = TestFactory.create_test("write")

test1.run()
test2.run()

```

Output:

```
Running NVMe Read Test
Running NVMe Write Test
```

This design makes it easier to add new test classes and centralize object creation logic.

### 7. Advantages and disadvantages

|Advantages|Disadvantages|
|---|---|
|Centralizes object creation|Introduces additional abstraction|
|Reduces coupling between client code and concrete classes|Factory must be maintained as classes are added|
|Makes code easier to extend|Can be unnecessary for very small programs|
|Simplifies object initialization|Complex factories can become difficult to maintain|
|Supports polymorphism|Does not automatically enforce a common interface|

### 8. Simple Factory vs. Factory Method

There is an important distinction between these two concepts.

Simple Factory is the approach demonstrated above. One factory method selects a concrete class based on an input, such as `"dog"` or `"cat"`.

Factory Method is a formal Gang of Four (GoF) design pattern. It defines a creation method that subclasses can override to decide which concrete object to instantiate.

|Feature|Simple Factory|Factory Method|
|---|---|---|
|Object selection|Typically through conditions or a dictionary|Through subclass implementations|
|Inheritance required|No|Typically yes|
|Complexity|Low|Moderate|
|Best for|Centralized selection of object types|Extensible frameworks with customizable object creation|

Key takeaway: The Factory pattern separates object creation from object usage. Your application requests an object from a factory, and the factory handles selecting and instantiating the appropriate class.

For learning Python design patterns, start with the Simple Factory implementation, then move to Factory Method and Abstract Factory.

