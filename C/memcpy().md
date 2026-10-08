The **`memcpy` function** in C ==copies a specified block of raw memory from a source address to a destination address==. It is defined in the `<string.h>` header and performs a fast, byte-by-byte binary copy without any type checking, boundary validation, or interpretation of the underlying data.

syntax
```c
#include <string.h>

void *memcpy(void *restrict dest, const void *restrict src, size_t n);
```

Parameters and Return Value

- **`dest`:** A `void*` pointer to the destination memory block where data will be copied. [1](https://www.geeksforgeeks.org/cpp/memcpy-in-cc/), [2](https://www.tutorialspoint.com/c_standard_library/c_function_memcpy.htm)
- **`src`:** A `const void*` pointer to the source memory block. [1](https://www.scaler.com/topics/memcpy-in-c/), [2](https://www.geeksforgeeks.org/cpp/memcpy-in-cc/)
- **`n`:** The exact number of **bytes** to copy (`size_t`). [1](https://www.ibm.com/docs/en/zos/3.1.0?topic=functions-memcpy-copy-buffer), [2](https://www.tutorialspoint.com/c_standard_library/c_function_memcpy.htm)
- **Return Value:** It returns a `void*` pointer to the original destination address (`dest`)

##### Example 1:
```c
#include <stdio.h>
#include <string.h>

int main() {
    int source[] = {10, 20, 30, 40, 50};
    int destination[5];

    // Copying 5 integer elements (5 * 4 bytes = 20 bytes on most systems)
    memcpy(destination, source, sizeof(source));

    for(int i = 0; i < 5; i++) {
        printf("%d ", destination[i]); // Prints: 10 20 30 40 50
    }
    return 0;
}
```

##### Example 2: 
```c
#include <stdio.h>
#include <string.h>

struct Point {
    int x;
    int y;
};

int main() {
    struct Point p1 = {15, 25};
    struct Point p2;

    // Shallow copy of the structure's bytes
    memcpy(&p2, &p1, sizeof(struct Point));

    printf("p2.x = %d, p2.y = %d\n", p2.x, p2.y); // Prints: p2.x = 15, p2.y = 25
    return 0;
}
```

