
`memset()` is a standard C library function that **fills a block of memory with a specified byte value**. It is declared in `<string.h>`.
### Syntax

```c
#include <string.h>

void *memset(void *ptr, int value, size_t num);
```

### Parameters

- **`ptr`**: Pointer to the beginning of the memory block to fill.
- **`value`**: The value to write. Only the **lowest 8 bits** (one byte) are used.
- **`num`**: Number of bytes to fill.
    
### Return Value

Returns the original pointer (`ptr`).
## Example 1: Initialize an array to zero

```c
#include <stdio.h>
#include <string.h>

int main() {
    int array[5];

    memset(array, 0, sizeof(array));

    for (int i = 0; i < 5; i++)
        printf("%d ", array[i]);

    return 0;
}
```

Output:

```
0 0 0 0 0
```

Since all bytes are set to `0x00`, each integer becomes `0`.

## Example 2: Fill a character buffer

```c
char buffer[10];

memset(buffer, 'A', sizeof(buffer));
```

Memory contents:

```
A A A A A A A A A A
```

In hexadecimal:

```
41 41 41 41 41 41 41 41 41 41
```

## Example 3: Fill with 0xFF

```c
unsigned char data[8];

memset(data, 0xFF, sizeof(data));
```

Memory:

```
FF FF FF FF FF FF FF FF
```


# Important: `memset()` operates on **bytes**

Suppose you have:

```c
int x[4];
```

Each `int` is typically 4 bytes.

```
Before:

+----+----+----+----+
| ?? | ?? | ?? | ?? |
+----+----+----+----+
```

Calling

```c
memset(x, 0, sizeof(x));
```

writes

```
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
```

Every byte becomes zero.

# Common mistake

Many beginners try this:

```c
int array[5];

memset(array, 1, sizeof(array));
```

They expect

```
1 1 1 1 1
```

But that's **not** what happens.

Each byte becomes:

```
01 01 01 01
```

Each integer becomes

```
0x01010101
```

which is

```
16843009
```

So

```c
printf("%d\n", array[0]);
```

prints something like

```
16843009
```

not `1`.

# When is `memset()` safe for integers?

These are safe:

```c
memset(array, 0, sizeof(array));
```

because all-zero bytes represent integer zero.

```c
memset(array, -1, sizeof(array));
```

is commonly used on systems that use two's complement representation because each byte becomes `0xFF`, making each integer `-1`. However, for maximum portability, assigning `-1` in a loop is preferred.

Anything else (such as `1`, `2`, `5`, etc.) should **not** be used to initialize integer arrays.


# Typical uses of `memset()`

### Zero a structure

```c
struct nvme_command cmd;

memset(&cmd, 0, sizeof(cmd));
```

This is very common in NVMe, PCIe, and Linux kernel programming because it ensures all fields start in a known state.

### Clear a buffer

```c
char buffer[4096];

memset(buffer, 0, sizeof(buffer));
```


### Initialize a DMA buffer

```c
void *dma_buffer = malloc(4096);

memset(dma_buffer, 0, 4096);
```


### Set memory to a pattern

```c
memset(buffer, 0xAA, size);
```

Useful for debugging or memory tests.


# How `memset()` works internally

Conceptually, it performs something like:

```c
void *my_memset(void *ptr, int value, size_t n)
{
    unsigned char *p = ptr;

    while (n--) {
        *p++ = (unsigned char)value;
    }

    return ptr;
}
```

Real implementations are much more optimized. They often write multiple bytes (e.g., 8, 16, 32, or 64 bytes) at a time using CPU-specific instructions such as SIMD operations for higher performance.

## Relation to NVMe programming

In NVMe development, `memset()` is used frequently to initialize command structures before setting only the required fields:

```c
struct nvme_passthru_cmd cmd;

memset(&cmd, 0, sizeof(cmd));

cmd.opcode = NVME_ADMIN_IDENTIFY;
cmd.nsid = 1;
cmd.addr = (__u64)buffer;
cmd.data_len = 4096;
cmd.cdw10 = 1;
```

Zero-initializing the structure ensures all reserved and unused fields contain valid values, helping prevent unexpected controller behavior or kernel errors.