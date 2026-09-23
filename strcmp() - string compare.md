
In C, the **`strcmp()` function** is a built-in library function used to **compare two strings character-by-character** based on their ASCII values. It is defined in the `<string.h>` header file.

You cannot use the standard equality operator (`==`) to compare string contents in C because `==` only compares the memory addresses of the character arrays, not the actual text.

```c
#include <string.h>

int strcmp(const char *str1, const char *str2);
```

Return Values

The function evaluates the strings lexicographically (in dictionary order) and returns an integer:

- **`0`**: The strings are **identical**
- **`< 0` (Negative value)**: The first string (`str1`) is **less than** the second string (`str2`). This happens if the first non-matching character in `str1` has a lower ASCII value than the corresponding character in `str2`
- **`> 0` (Positive value)**: The first string (`str1`) is **greater than** the second string (`str2`). This happens if the first non-matching character in `str1` has a higher ASCII value

```c
#include <stdio.h>
#include <string.h> // Required header for strcmp

int main() {
    char str1[] = "Apple";
    char str2[] = "Apple";
    char str3[] = "Banana";

    // Scenario 1: Strings are equal
    if (strcmp(str1, str2) == 0) {
        printf("str1 and str2 are identical.\n");
    }

    // Scenario 2: str1 is lexicographically less than str3
    if (strcmp(str1, str3) < 0) {
        printf("\"%s\" comes before \"%s\" in the dictionary.\n", str1, str3);
    }

    // Scenario 3: str3 is lexicographically greater than str1
    if (strcmp(str3, str1) > 0) {
        printf("\"%s\" comes after \"%s\" in the dictionary.\n", str3, str1);
    }

    return 0;
}
```

- **Case Sensitivity:** `strcmp()` is case-sensitive. For instance, `"Apple"` and `"apple"` are not equal because uppercase `'A'` (ASCII 65) and lowercase `'a'` (ASCII 97) have different values
- **Safety Warning:** `strcmp()` assumes both strings are null-terminated (`'\0'`). If you pass a character array that lacks a null terminator, it will continue reading memory past the buffer boundaries, which leads to undefined behavior. If you want to limit the comparison to a specific number of characters, use `strncmp()` instead
- 

