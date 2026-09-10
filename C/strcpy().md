
In C programming, the **`strcpy()` function is used to copy the entire contents of a source string (including the terminating null character `\0`) into a destination character array**. [1](https://www.geeksforgeeks.org/c/strcpy-in-c/), [2](https://www.programiz.com/c-programming/library-function/string.h/strcpy)

Because strings in C are arrays of characters rather than primitive data types, you cannot use the standard assignment operator (`=`) to assign or copy a string after it has been declared. `strcpy()` is the standard tool provided in the `<string.h>` library to overcome this limitation.

syntax
```c
#include <string.h>

char *strcpy(char *dest, const char *src);
```

#### Example
```c
#include <stdio.h>
#include <string.h>

int main() {
    char source[] = "Hello, World!";
    // Ensure the destination array is large enough to hold the source + null terminator
    char destination[20]; 

    // Copying source to destination
    strcpy(destination, source);

    printf("Source: %s\n", source);
    printf("Destination: %s\n", destination);

    return 0;
}
```


