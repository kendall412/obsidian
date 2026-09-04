
## User Defined

### Statically Allocated Buffer (Fixed Size)

You can create your own buffers to store data. These are typically implemented using arrays or dynamically allocated blocks of memory. [1](https://www.reddit.com/r/C_Programming/comments/1h6yt8i/what_exactly_is_a_buffer/)

Statically Allocated Buffer (Fixed Size)

An array acts as a fixed-size buffer. You must ensure you don't write more data than the array can hold, or you will cause a dangerous memory error known as a **buffer overflow**

```c
#include <stdio.h>
#include <string.h>

int main() {
    // A character buffer that can hold up to 19 characters + 1 null terminator
    char buffer[20]; 

    // Safe copying: limits data to the size of the buffer
    strncpy(buffer, "Hello, World!", sizeof(buffer) - 1);
    buffer[sizeof(buffer) - 1] = '\0'; // Ensure null-termination

    printf("Buffer contains: %s\n", buffer);
    return 0;
}
```

### Dynamically Allocated Buffer

Use [[malloc()]] or calloc