#typedef
The **`typedef` keyword** in C is used to create a **new name (an alias)** for an existing data type. It does not introduce a brand-new data type; rather, it provides a shorter, more descriptive nickname for types you already use, making your code cleaner and more maintainable.

### 1. General Syntax

The standard syntax mimics a normal variable declaration, but with the word `typedef` placed at the front.

```c
typedef existing_type new_name;
```

Once defined, you use `new_name` exactly like any built-in C type: [1](https://techvidvan.com/tutorials/c-typedef-with-examples/), [2](https://flaviocopes.com/c-typedef/)

```c
typedef unsigned long ulong; // Create the alias 'ulong'
ulong distance = 500000;     // Equivalent to: unsigned long distance = 500000;
```

### 2. Common Use Cases

A. Simplifying Structures (`struct`)

By default, declaring a structure variable requires writing out the `struct` keyword every time. `typedef` eliminates this repetition.

**Without `typedef`:**
```c
struct Point {
    int x;
    int y;
};

struct Point p1; // Must include the word 'struct'
```

**With `typedef`:**
```c
typedef struct {
    int x;
    int y;
} Point;

Point p1; // Much cleaner!
```