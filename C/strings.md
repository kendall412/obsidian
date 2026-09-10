There are several ways to initialize strings in C

#### 1. Using a String Literal (Most Common)
The easiest way to initialize a string is to use a **string literal** inside double quotes. The compiler automatically calculates the required memory size and adds the terminating `'\0'` character at the end

```c
// The compiler automatically sets the size to 6 bytes (5 for "Hello" + 1 for '\0')
char str1[] = "Hello"; 
```

#### 2. Specifying an Array Size
You can explicitly define the size of the character array. Ensure the size is large enough to hold all characters plus the null terminator. If the string literal is shorter than the allocated array size, the remaining elements are automatically filled with zeros (`'\0'`). [1](https://www.youtube.com/watch?v=cnfRyvo41Bs), [2](https://www.classes.cs.uchicago.edu/archive/2020/winter/15200-1/lecs/notes/Lec10Strings.html), [3](https://learn.microsoft.com/en-us/cpp/c-language/initializing-strings?view=msvc-170), [4](https://www.youtube.com/watch?v=m4wVJuaQu_4&vl=en), [5](https://www.youtube.com/watch?v=60OI5tzmkCw), [6](https://www.geeksforgeeks.org/c/strings-in-c/)

```c
// Allocates 50 bytes, fills the first 6 bytes with "Hello\0", and zeros out the rest
char str2[50] = "Hello"; 
```

#### 3. Character-by-Character Initialization
ou can initialize a string like a standard array using curly braces `{}` and single quotes for individual characters. When using this method, **you must explicitly include the `'\0'` null terminator** at the end.

```c
char str3[] = {'H', 'e', 'l', 'l', 'o', '\0'};
```

#### 4. Using a Character Pointer
Instead of an array, you can point directly to a string literal using a pointer. This stores the string in a **read-only section of memory**. Because the contents cannot safely be changed, it is best practice to prefix this with `const`.

```c
// Points to a read-only memory location; the string cannot be modified
const char *str4 = "Hello"; 
```