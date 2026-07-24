
**Byte alignment** (or memory alignment) is the practice of **arranging data in computer memory at addresses that are multiples of the data's size** or the system's word size. [1](https://medium.com/@pawanwagh/understanding-memory-alignment-for-better-performance-3075787cfd3b), [2](https://www.geeksforgeeks.org/dsa/data-structure-alignment-how-data-is-arranged-and-accessed-in-computer-memory/)

Modern computer processors do not read from or write to physical memory one individual byte at a time. Instead, they fetch memory in fixed chunks called words or cache lines (typically 4, 8, 32, or 64 bytes wide). Because of this hardware architecture, alignment ensures that data can be accessed efficiently in the minimum number of memory cycles. [1](https://stackoverflow.com/questions/2846914/what-is-meant-by-memory-is-8-bytes-aligned), [2](https://www.youtube.com/watch?v=H29YPfj0KCU&t=13), [3](https://www.youtube.com/shorts/4lhyo1Ce-GQ), [4](https://medium.com/@pawanwagh/understanding-memory-alignment-for-better-performance-3075787cfd3b)

### How Alignment Works

A piece of data is considered **n-byte aligned** if its memory address is evenly divisible by n (where n is a power of 2). For example: [1](https://en.wikipedia.org/wiki/Data_structure_alignment)

- **1-byte alignment**: Can be stored at any memory address (e.g., `0x00`, `0x01`, `0x02`).
- **2-byte alignment**: Must be stored at an even memory address (e.g., `0x00`, `0x02`, `0x04`).
- **4-byte alignment**: Address must end in `0`, `4`, `8`, or `C` in hexadecimal (e.g., `0x00`, `0x04`, `0x08`).
- **8-byte alignment**: Address must end in `0` or `8` in hexadecimal (e.g., `0x00`, `0x08`, `0x10`). [1](https://medium.com/@pawanwagh/understanding-memory-alignment-for-better-performance-3075787cfd3b), [2](https://www.youtube.com/watch?v=0y3k8VCNGmU), [3](https://www.eventhelix.com/embedded/byte-alignment-and-ordering/), [4](https://web.mit.edu/rust-lang_v1.25/arch/amd64_ubuntu1404/share/doc/rust/html/reference/type-layout.html)

### Why Byte Alignment Matters

1. Performance and Efficiency

When data is aligned, the CPU can usually retrieve it with a single memory read. If a 4-byte integer is "misaligned"—for example, split across a boundary at address `0x03`—the CPU must perform **two separate memory reads** to piece the integer together. It must grab the first byte from the first row, the remaining three bytes from the second row, and then combine them in a register. This introduces a massive performance penalty. [1](https://www.youtube.com/watch?v=H29YPfj0KCU&t=13), [2](https://softwareengineering.stackexchange.com/questions/441478/memory-alignment), [3](https://medium.com/@pawanwagh/understanding-memory-alignment-for-better-performance-3075787cfd3b)

2. Hardware Constraints

Some processor architectures (like certain RISC or ARM chips) cannot handle misaligned data at all. Attempting to read a misaligned address on these systems will trigger a hardware exception or CPU fault, causing the entire application to crash.

3. Advanced Vector Instructions

Modern processors use specialized SIMD (Single Instruction Multiple Data) instruction sets like AVX or SSE to perform math on large groups of numbers simultaneously. These instructions frequently mandate strict alignment boundaries (like 16-byte or 32-byte alignment) to function.

### Data Structure Padding

To ensure proper alignment, compilers like those for C, C++, and Rust automatically insert invisible, empty bytes called **padding** between variables inside a struct. [1](https://medium.com/@pawanwagh/understanding-memory-alignment-for-better-performance-3075787cfd3b), [2](https://www.youtube.com/watch?v=OS8g4bOfVU4&t=11)

Consider this conceptual C/C++ struct:

```
struct Example {
    char a;   // 1 byte
    int b;    // 4 bytes
};
```

Naively, you might think this struct takes up 5 bytes (1 + 4). However, the compiler enforces a 4-byte alignment rule for the `int`. The actual layout in memory looks like this:

| Address Offset | Data Component | Status                    |
| -------------- | -------------- | ------------------------- |
| 0x00           | char a         | filled                    |
| 0x01           | Padding Byte   | Wasted Space              |
| 0x02           | Padding Byte   | Wasted Space              |
| 0x03           | Padding Byte   | Wasted Space              |
| 0x04 - 0x07    | int b          | Filled (Properly Aligned) |
