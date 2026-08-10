#file_descriptor #fd

# General

A **file descriptor** is a small integer that a program uses to refer to an open file or I/O resource. It is commonly used by operating systems like Linux, macOS, and Unix.

Examples of things represented by file descriptors:

- A file
- A terminal
- A network socket
- A pipe
- A device

When a program opens something, the operating system returns a number called a file descriptor.
Example in C/Python-like terms:

```text
open file.txt  →  returns file descriptor 3
```

Then the program can say:

```text
read from descriptor 3
write to descriptor 3
close descriptor 3
```

The program does not directly access the file itself. It asks the operating system to perform operations using that descriptor.

Common file descriptors:

```text
0 = standard input  stdin
1 = standard output stdout
2 = standard error  stderr
```

For example:

```bash
echo hello
```

writes to file descriptor `1`, which is usually your terminal. Errors usually go to file descriptor `2`.

Simple analogy:

A file descriptor is like a **ticket number** or **handle**. You do not carry the actual file around. You have a number that tells the operating system:

> “Use the open file/resource associated with this number.”

Example in Python:

```python
f = open("data.txt", "r")
print(f.fileno())
```

Output might be:

```text
3
```

That means Python’s file object is using file descriptor `3` internally.

Then when you close the file:

```python
f.close()
```

the file descriptor is released.

# Python

`os.open()` **opens a file descriptor for a low-level, operating-system-dependent file handling operation.** [1](https://askubuntu.com/questions/1253742/python-os-open-is-not-working-in-ubuntu), [2](https://zetcode.com/python/os-open/), [3](https://docs.python.org/3/library/os.html)

Unlike Python's standard `open()` function which returns a high-level file object, `os.open()` interacts directly with the operating system kernel and returns a standard **integer file descriptor**. [1](https://www.digitalocean.com/community/tutorials/python-io-bytesio-stringio), [2](https://stackoverflow.com/questions/77260443/why-am-i-getting-int-as-a-return-value-from-os-open)

Key Differences: `os.open()` vs. `open()`

|Feature|`os.open()`|Built-in `open()`|
|---|---|---|
|**Return Type**|Integer file descriptor|File-like object (`io.TextIOWrapper`)|
|**Abstraction Level**|Low-level (OS-dependent)|High-level (Python-abstracted)|
|**Flags**|Explicit bitwise masks (`os.O_RDONLY`)|Simple mode strings (`'r'`, `'w'`)|
|**Use Case**|System programming, device locks|Everyday file reading and writing|
Common Flags

To specify how to open a file, you pass bitwise flags combined with the `|` operator: [1](https://www.codewithc.com/python-how-to-read-a-file-simplifying-file-operations-2/)

- **`os.O_RDONLY`**: Open for reading only.
- **`os.O_WRONLY`**: Open for writing only.
- **`os.O_RDWR`**: Open for reading and writing.
- **`os.O_CREAT`**: Create the file if it does not exist.
- **`os.O_EXCL`**: Fail if the file already exists (used with `os.O_CREAT`). [1](https://www.ibm.com/docs/en/zos/3.1.0?topic=commands-open), [2](https://www.geeksforgeeks.org/python/python-os-open-method/), [3](https://linux.die.net/man/2/open), [4](https://man7.org/linux/man-pages/man2/open.2.html), [5](https://perldoc.perl.org/functions/sysopen)

### Example

```python
import os

# Open a file for writing, create it if missing, clear it if it exists
flags = os.O_WRONLY | os.O_CREAT | os.O_TRUNC
fd = os.open("example.txt", flags)

# Write bytes directly using the integer file descriptor
os.write(fd, b"Hello from os.open!")

# Always close the file descriptor manually
os.close(fd)
```

