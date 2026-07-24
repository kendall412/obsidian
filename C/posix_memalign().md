
`posix_memalign()` is a POSIX standard library function that **allocates memory with a specified  [[byte alignment]]**. Unlike `malloc()`, which only guarantees alignment suitable for normal data types, `posix_memalign()` lets you request a specific memory boundary (e.g., 64 bytes, 4096 bytes).

It is commonly used in:

- NVMe applications
- DMA (Direct Memory Access)
- PCIe drivers
- High-performance networking
- SIMD (SSE/AVX) programming
- Storage applications like SPDK
    
It is declared in:

```c
#include <stdlib.h>
```

# Syntax

```c
int posix_memalign(void **memptr,
                   size_t alignment,
                   size_t size);
```

# Parameters

### 1. `memptr`

Pointer to a pointer.

```c
void *buffer;
```

You pass its address:

```c
&buffer
```

After the function succeeds,

```c
buffer
```

points to the allocated memory.

### 2. `alignment`

Specifies the required memory alignment.

Examples:

```text
8 bytes
16 bytes
32 bytes
64 bytes
512 bytes
4096 bytes
```

Requirements:

- Must be a power of two.
- Must be a multiple of `sizeof(void *)`.

For example, on a 64-bit machine:

```text
sizeof(void *) = 8
```

Valid alignments:

```text
8
16
32
64
128
256
512
1024
2048
4096
```

Invalid:

```text
3
20
100
```

### 3. `size`

Number of bytes to allocate.

Example:

```c
4096
```

allocates one page.

# Return value

Unlike `malloc()`, which returns a pointer, `posix_memalign()` returns an error code.

Success:

```c
0
```

Failure:

```text
EINVAL
ENOMEM
```


# Example

```c
#include <stdio.h>
#include <stdlib.h>

int main()
{
    void *buffer;

    int ret = posix_memalign(&buffer, 4096, 4096);

    if (ret != 0) {
        printf("Allocation failed\n");
        return 1;
    }

    printf("Buffer = %p\n", buffer);

    free(buffer);

    return 0;
}
```

Output might be:

```text
Buffer = 0x7f3ac5600000
```

Notice:

```text
...0000
```

The address is divisible by 4096.

---

# Why not just use `malloc()`?

Suppose:

```c
char *buffer = malloc(4096);
```

The returned address could be:

```text
0x7f3512ab6038
```

This address is **not necessarily** aligned to a 4096-byte boundary.

For many applications that's perfectly fine.

For DMA or NVMe, it may not be.

---

# Visual example

Normal `malloc()`:

```
Memory

0x1000 --------------------
0x1001
0x1002
0x1003
0x1004
...
0x1038 <-- malloc() returned here
```

Not page aligned.

---

`posix_memalign(...,4096,...)`

```
Memory

0x1000
0x2000  <-- returned pointer
0x3000
0x4000
```

Always aligned on a page boundary.

---

# Why alignment matters

Modern CPUs fetch memory in blocks called **cache lines**.

Typical cache line:

```text
64 bytes
```

If your structure begins on a cache-line boundary:

```
| Cache Line 0 |
+-------------------------------+
| structure begins here         |
+-------------------------------+
```

the CPU can often access it more efficiently.

If it spans cache lines:

```
| Cache Line 0 |
+----------------------+
           |
           v
+----------------------+
| Cache Line 1         |
```

the CPU may need two memory accesses instead of one.

---

# Why NVMe uses aligned memory

Consider an NVMe Identify command:

```
Host

Identify Controller

      |
      v

4096-byte buffer
```

The controller DMA engine transfers:

```
SSD Controller
      |
      | DMA
      |
      v

Host RAM
```

Many drivers and user-space frameworks allocate the buffer with page alignment:

```c
posix_memalign(&buffer, 4096, 4096);
```

Benefits include:

- Page-aligned buffers simplify DMA mapping.
    
- Better compatibility with operating system memory management.
    
- Improved performance in many storage stacks.
    

---

# Example in NVMe

```c
struct nvme_passthru_cmd cmd;

void *buffer;

posix_memalign(&buffer, 4096, 4096);

memset(buffer, 0, 4096);

cmd.addr = (unsigned long long)buffer;
cmd.data_len = 4096;

ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);

free(buffer);
```

Here:

1. Allocate a page-aligned buffer.
    
2. Clear it with `memset()`.
    
3. Pass its address to the NVMe driver.
    
4. The controller performs DMA into that buffer.
    
5. Free the memory when finished.
    

---

# Comparison with other allocation functions

|Function|Alignment|Standard|Notes|
|---|---|---|---|
|`malloc()`|Suitable for any fundamental type|C standard|General-purpose allocation|
|`calloc()`|Same as `malloc()`|C standard|Allocates and zero-initializes memory|
|`realloc()`|Same as `malloc()`|C standard|Resizes an existing allocation|
|`posix_memalign()`|User-specified alignment|POSIX|Returns an error code; memory is freed with `free()`|
|`aligned_alloc()`|User-specified alignment|C11|Requires `size` to be a multiple of `alignment`|

---

## When should you use `posix_memalign()`?

Use it when you need a buffer with a guaranteed alignment, such as:

- DMA transfers (including many NVMe use cases)
    
- Page-aligned memory (e.g., 4096-byte alignment)
    
- Cache-line alignment (e.g., 64 bytes)
    
- SIMD instructions (SSE, AVX, AVX-512)
    
- High-performance I/O frameworks like SPDK
    

For ordinary application code that doesn't have alignment requirements, `malloc()` or `calloc()` is usually sufficient.