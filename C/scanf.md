
In C programming, **`scanf()`** is a standard library function used to **read formatted data from the standard input stream** (usually the keyboard). It is defined in the `<stdio.h>` header file. [1](https://www.geeksforgeeks.org/c/scanf-in-c/)

basic syntax

```c
scanf("format_string", &variable1, &variable2, ...);
```

- **Format String:** Contains format specifiers (like `%d` or `%f`) indicating the type of data to expect.

- **The Address-of Operator (`&`):** You must pass the **memory address** of the variables (using `&`) so `scanf` can directly modify their values. _(Exception: String arrays do not need `&` because their names already act as addresses)._ [1](https://www.youtube.com/watch?v=RAbwchpvO1s&t=37), [2](https://www.youtube.com/watch?v=ZZexryh1M3A), [3](https://www.youtube.com/watch?v=xedk5KXg0VI&t=193), [4](https://www.youtube.com/watch?v=9-hBZwePrPg&vl=en), [5](https://www.geeksforgeeks.org/c/scanf-in-c/)


Common Format Specifiers

| Specifier | Data Type                         | Example                    |
| --------- | --------------------------------- | -------------------------- |
| **`%d`**  | Integer (`int`)                   | `scanf("%d", &myInt);`     |
| **`%f`**  | Floating-point (`float`)          | `scanf("%f", &myFloat);`   |
| **`%lf`** | Double-precision float (`double`) | `scanf("%lf", &myDouble);` |
| **`%c`**  | Single character (`char`)         | `scanf("%c", &myChar);`    |
| **`%s`**  | String of characters (`char[]`)   | `scanf("%s", myString);`   |

#### Example 
```c
#include <stdio.h>

int main() {
    int age;
    float height;

    printf("Enter your age and height: ");
    // Reads two inputs separated by whitespace
    scanf("%d %f", &age, &height); 

    printf("Age: %d, Height: %.2f\n", age, height);
    return 0;
}

```