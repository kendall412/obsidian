
> The **Submission Queue Head Pointer (SQ Head)** in NVMe is a pointer maintained by the **controller** that indicates **the next command entry in the Submission Queue (SQ) to be processed**.

---
# 🔹 Where It Fits

NVMe Submission Queue is a circular buffer:

```
Host writes commands → advances SQ Tail
Controller consumes → advances SQ Head
```

- **SQ Tail (host-owned)** → where new commands are posted
- **SQ Head (controller-owned)** → which commands have been consumed

# 🔹 Definition

> **SQ Head Pointer = index of the next SQ entry the controller will read/process**

# 🔹 How It Works

## ✔ Step-by-step

### 1. Host submits commands

```
Write SQE at Tail
Tail++
Ring doorbell
```

### 2. Controller processes commands

```
Reads SQE at Head
Executes command
Head++
```

# 🔹 Circular Behavior

```
Head = (Head + 1) % Queue Size
```

# 🔹 Example

Queue size = 8

```
Initial:
Head = 0, Tail = 0

Host submits 3 commands:
Tail = 3

Controller processes 2:
Head = 2
```

Queue state:

```
Index:   0   1   2   3   4   5   6   7
        [x] [x] [ ] [ ] [ ] [ ] [ ] [ ]
         ↑       ↑
       done    next to process
       (Head=2)
```

# 🔹 Where SQ Head Is Reported

The host does **not directly read SQ Head from memory**.

Instead, it is returned in the **Completion Queue Entry (CQE)**:

```
CQE DW2:
  bits 15:0 → SQ Head Pointer (SQHD)
  bits 31:16 → SQ Identifier
```

👉 This tells the host:

> “I have consumed commands up to this point”

# 🔹 Why SQ Head Matters

## ✔ Flow Control

- Prevents overwriting unprocessed entries

```
Available space = (Queue Size - (Tail - Head))
```

## ✔ Synchronization

- Host knows which commands are safe to overwrite

## ✔ Performance

- Enables lock-free producer/consumer model

# 🔹 Relationship with [[doorbell]]

- Host updates **SQ Tail Doorbell**
- Controller updates **SQ Head internally**
- Host learns SQ Head via CQE

---

# 🔹 Key Insight

> The host produces commands, the controller consumes them—SQ Head is the controller’s progress marker.

# 🔹 One-Line Summary

**The Submission Queue Head Pointer is the controller-maintained index of the next command to process in the SQ, reported back to the host via completion entries to enable flow control and synchronization.**

---

If you want, I can next:

- show **exact CQE DW2 bit-level layout**
- or simulate **head/tail wraparound with phase tags step-by-step**