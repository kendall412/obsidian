
You **do not have to cast `ptr` itself** just because `malloc()` returns a `void *`. The important distinction is between the **type of the variable** and the **type of the expression being assigned to it**.

Suppose you have:

```c
int *ptr;
```

This declares `ptr` as an `int *`. It means:

> `ptr` is a variable capable of storing the address of an `int`.

Now consider:

```c
ptr = malloc(n * sizeof(int));
```

`malloc()` returns:

```c
void *
```

So conceptually you have:

```text
           malloc(...)
               │
               ▼
             void *
               │
               │ automatic conversion in C
               ▼
             int *
               │
               ▼
              ptr
```

Because `ptr` is already declared as `int *`, C automatically converts the `void *` returned by `malloc()` to `int *`.

So **no cast is necessary in C**.

### Then what does `(int *)` do?

When you write:

```c
ptr = (int *)malloc(n * sizeof(int));
```

the cast applies to the **result of `malloc()`**, not to `ptr`.

Break it apart:

```c
malloc(n * sizeof(int))
```

returns:

```c
void *
```

Then:

```c
(int *) malloc(n * sizeof(int))
```

explicitly converts that result:

```text
void *
  │
  │ (int *)
  ▼
int *
```

Then the `int *` is assigned to `ptr`:

```text
malloc(...)
    │
    ▼
  void *
    │
    │ explicit cast
    ▼
  int *
    │
    ▼
   ptr
```

But since C already performs that conversion automatically, these are effectively equivalent:

```c
int *ptr;

ptr = malloc(n * sizeof(int));
```

and:

```c
int *ptr;

ptr = (int *)malloc(n * sizeof(int));
```

### Think of the declaration and assignment separately

This is the part that often causes confusion.

```c
int *ptr;
```

does **not** mean that every value assigned to `ptr` is already an `int *`.

It means:

> "`ptr` is a variable whose type is `int *`."

The expression on the right-hand side still has its own type:

```c
ptr = malloc(100);
      └─────────┘
         void *
```

C sees:

```text
Left side                 Right side

ptr                       malloc(100)
 │                            │
 ▼                            ▼
int *                        void *

          assignment
int *  <--------------  void *
              ↑
       C allows this conversion
       automatically
```

That's why the normal C style is simply:

```c
int *ptr = malloc(n * sizeof *ptr);
```

No `(int *)` is needed.

One subtle point: **C++ is different**. C++ does not allow this implicit `void *` → `int *` conversion, which is one reason you sometimes see `(int *)malloc(...)` in code.