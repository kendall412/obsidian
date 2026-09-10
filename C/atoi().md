
In C, **`atoi` stands for ASCII to Integer**. It is a built-in library function used to **convert a string of characters representing a number into an actual integer (`int`) value**.

The function is defined within the **`<stdlib.h>`** header file.

syntax
```c
#include <stdlib.h>

int atoi(const char *str);
```

#### Example
```c
#include <stdio.h>
#include <stdlib.h> // Required for atoi

int main() {
    char str1[] = "1234";
    char str2[] = "5678";
    
    // Convert strings to integers
    int num1 = atoi(str1);
    int num2 = atoi(str2);
    
    // You can now perform mathematical operations
    int sum = num1 + num2;
    
    printf("The sum is: %d\n", sum); // Output: The sum is: 6912
    
    return 0;
}
```

