#c 

The **`fread()` function** in C is a standard library function declared in `<stdio.h>` that is used to **read blocks of binary data** from a file stream and store them into a memory buffer. It transfers the exact binary representation from disk into memory without any data conversion, making it highly efficient for reading arrays, structures, and block data.

#### syntax
```c
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
```

- **`ptr`**: Pointer to the block of memory (buffer) where the retrieved data will be stored.
- **`size`**: The size, in bytes, of each individual element to be read.
- **`nmemb`**: The number of elements to read.
- **`stream`**: Pointer to the `FILE` object representing the open file stream


#### Example
```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int id;
    char name[20];
    float salary;
} Employee;

int main() {
    // 1. Open the file in binary read mode ("rb")
    FILE *fp = fopen("employees.bin", "rb");
    if (fp == NULL) {
        perror("Error opening file");
        return EXIT_FAILURE;
    }

    Employee list[3];
    int expected_count = 3;

    // 2. Read 3 Employee structures from the file
    size_t elements_read = fread(list, sizeof(Employee), expected_count, fp);

    printf("Elements requested: %d | Elements successfully read: %zu\n", expected_count, elements_read);

    // 3. Check if we read fewer elements than requested
    if (elements_read < expected_count) {
        if (feof(fp)) {
            printf("End of file reached prematurely.\n");
        } else if (ferror(fp)) {
            printf("An error occurred while reading the file.\n");
        }
    }

    // Print successfully read data
    for (size_t i = 0; i < elements_read; i++) {
        printf("ID: %d, Name: %s, Salary: %.2f\n", list[i].id, list[i].name, list[i].salary);
    }

    // 4. Close the file stream
    fclose(fp);
    return EXIT_SUCCESS;
}
```
