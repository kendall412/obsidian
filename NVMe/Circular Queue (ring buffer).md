
A **circular queue (ring buffer)** is a fixed-size queue where the **end wraps back to the beginning**, so the storage is reused continuously instead of shifting elements.

# 🔹 Core Idea

```
Linear queue:
[ A ][ B ][ C ][ _ ][ _ ]

Circular queue:
[ A ][ B ][ C ][ _ ][ _ ]
   ↑                ↓
   └──── wrap ──────┘
```

👉 When the tail reaches the last slot, it **wraps to index 0**.

# 🔹 Key Pointers

A circular queue uses two indices:

- **Head** → where data is removed (consumer)
- **Tail** → where data is inserted (producer)

```
enqueue → write at tail, then tail++
dequeue → read at head, then head++
```

Both wrap using modulo arithmetic:

```
index = (index + 1) % queue_size
```

# 🔹 States

## ✔ Empty

```
head == tail   (and no valid data)
```

## ✔ Full (two common methods)

### Method 1 (leave one slot empty):

```
(head == (tail + 1) % size)
```

### Method 2 (explicit counter/flag):

- Track number of elements

# 🔹 Why Use Circular Queue

- **No data shifting** (unlike linear queues)
- **Constant time O(1)** operations
- Efficient memory reuse
- Ideal for **producer–consumer systems**


# 🔹 Real Example: NVMe Queues

NVMe uses circular queues for:

- **Submission Queues (SQ)**
- **Completion Queues (CQ)**

```
Host → writes commands to SQ (tail moves)
Controller → reads SQ (head moves)

Controller → writes completions to CQ (tail)
Host → reads CQ (head)
```

👉 Both sides operate independently using ring behavior.

# 🔹 Important NVMe Detail

Because NVMe queues are circular:

- Entries are reused after wraparound
- The system needs a way to distinguish:
    - **new vs old entries**

👉 This is why NVMe uses the **[[Phase Tag (P)]]**


# 🔹 Simple Analogy

Think of a **circular conveyor belt**:

- Items are placed at one point (tail)
- Removed at another (head)
- Belt keeps rotating—no need to reset

# 🔹 Visual Walkthrough

Queue size = 5
```
Initial:
[ _ ][ _ ][ _ ][ _ ][ _ ]
 H/T

After 3 inserts:
[ A ][ B ][ C ][ _ ][ _ ]
 H       T

After 2 removals:
[ _ ][ _ ][ C ][ _ ][ _ ]
       H       T

After wrap insert:
[ D ][ E ][ C ][ _ ][ _ ]
       H   T
```

# 🔹 One-Line Summary

A **circular queue** is a fixed-size queue where indices wrap around, enabling efficient, continuous reuse of memory without shifting data—fundamental to high-performance systems like NVMe.

