
The **`strpbrk()`** function in C searches a string for **any character** belonging to a specified set. It returns a **pointer** to the first character in the main string that matches one of the characters in the target set. The name stands for "**str**ing **p**ointer **br**eak", indicating where the string breaks upon finding a matching character.

#### Syntax

To use `strpbrk()`, you must include the `<string.h>` header file.

```c
#include <string.h>

char *strpbrk(const char *s1, const char *s2);
```

Parameters

- **`s1`**: The null-terminated C string to be scanned. 
- **`s2`**: The null-terminated C string containing the set of characters to search for.

Return Value

- Returns a **pointer** to the first occurrence of any character from `s2` within `s1`.
- Returns **`NULL`** if none of the characters from `s2` are found in `s1`.

#### Example
The following program searches for the first vowel in a sentence:
```c
#include <stdio.h>
#include <string.h>

int main() {
    const char *sentence = "Hello, World!";
    const char *vowels = "aeiouAEIOU";

    // Find the first vowel in the sentence
    char *result = strpbrk(sentence, vowels);

    if (result != NULL) {
        // %c prints the single character pointed to by result
        printf("First vowel found: %c\n", *result); 
        
        // %s prints the remaining substring from that pointer onward
        printf("Remainder of string: %s\n", result); 
    } else {
        printf("No vowels found.\n");
    }

    return 0;
}
```