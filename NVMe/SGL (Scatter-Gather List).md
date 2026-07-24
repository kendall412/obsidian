
> A **scatter–gather list (SGL)** in NVMe is a **[[data buffer]] description mechanism** that allows a command to reference **multiple, potentially non-contiguous memory regions** using a list of descriptors instead of a single contiguous buffer. In NVMe, an SGL descriptor is **embedded directly into the SQE at DW6–DW9**, with PSDT selecting SGL mode, allowing the controller to interpret those DWORDs as a **16-byte descriptor instead of PRP pointers**.

### Why SGL exists (context)

NVMe commands need to tell the controller _where the data lives in host memory_. There are two main mechanisms:

- **PRP (Physical Region Page)** — simpler, page-based
- **SGL (Scatter–Gather List)** — more flexible, descriptor-based

SGL is used when:

- Memory is fragmented (non-contiguous buffers)
- Large or complex I/O buffers are involved
- The host prefers descriptor chaining over fixed PRP structure

### Core idea

Instead of saying:

> “Here is one big contiguous buffer”

SGL says:

> “Here are multiple chunks of memory — go gather data from all of them (or scatter into them)”


### SGL structure (high level)

An SGL is made of **descriptors**, each typically 16 bytes, describing:

- Address of a memory segment
- Length of that segment
- Type of descriptor

### Common SGL descriptor types

1. **Data Block Descriptor**
    - Points directly to a data buffer
    - Contains:
        - Address
        - Length
2. **Segment Descriptor**
    - Points to another list of SGL descriptors (chaining)
3. **Last Segment Descriptor**
    - Same as segment but indicates end of chain
4. **Keyed Data Block (less common)**
    - Used with memory keys (e.g., RDMA scenarios)

---

### Example (conceptual)

Imagine an I/O buffer split across memory:
```
Buffer A → 4 KB at 0x1000
Buffer B → 8 KB at 0x9000
Buffer C → 4 KB at 0x20000
```

SGL would look like:
```
[Descriptor 1] → addr=0x1000, len=4K
[Descriptor 2] → addr=0x9000, len=8K
[Descriptor 3] → addr=0x20000, len=4K
```

Controller processes all entries sequentially.

---
### Where SGL appears in NVMe

In an NVMe command ([[SQ (Submission Queue)]] Entry), instead of PRP fields:

- The command includes:
    - **SGL Descriptor pointer (or inline descriptor)**
    - Controlled via:
        - **SGL flag in command**
        - **Controller capability (Identify Controller → SGL support)**

### PRP vs SGL (important distinction)

| Feature       | PRP                        | SGL                               |
| ------------- | -------------------------- | --------------------------------- |
| Structure     | Fixed (PRP1 + PRP2 + list) | Flexible descriptors              |
| Memory layout | Page-based                 | Arbitrary                         |
| Complexity    | Simple                     | More complex                      |
| Performance   | Good for simple I/O        | Better for complex/fragmented I/O |
| Use case      | Typical NVMe SSD access    | Advanced drivers, fabrics, RDMA   |

### When SGL is preferred

SGL is typically used in:

- **NVMe over Fabrics (NVMe-oF)** implementations
- Systems with:
    - IOMMU
    - Non-contiguous DMA mappings
- High-performance drivers optimizing memory usage

## Key takeaway

A **scatter–gather list in NVMe** is a **flexible DMA mapping mechanism** that allows a command to operate on **multiple discontiguous memory buffers via a chain of descriptors**, instead of requiring a contiguous memory region.

# SGL Descriptor Format (16 bytes)

An SGL descriptor consists of:

- **DW0–DW1 (64 bits)** → Address or pointer
- **DW2 (32 bits)** → Length
- **DW3 (32 bits)** → Type + subtype + control

### DW0–DW1: Address Field (64 bits)

```
Bits 63:0 → Address
```

- For **Data Block descriptors** → physical address of data buffer
- For **Segment descriptors** → pointer to another SGL list
- Must be **DMA-accessible**

### DW2: Length Field (32 bits)

```
## DW2: Length Field (32 bits)
```

Size of:
- Data buffer (for data descriptors), OR
- SGL segment (for segment descriptors)

### DW3: Descriptor Type and Control (32 bits)
```
Bits 31:24 → Reserved
Bits 23:20 → Subtype (ST)
Bits 19:16 → Type (TD)
Bits 15:0  → Type-specific field (often reserved or used for keyed SGL)
```

## Descriptor Type (TD field)

### Common values
|TD (bits 19:16)|Meaning|
|---|---|
|0x0|Data Block Descriptor|
|0x1|Bit Bucket Descriptor|
|0x2|Segment Descriptor|
|0x3|Last Segment Descriptor|
|0x4|Keyed Data Block Descriptor|
|others|Reserved|

# Subtype (ST field)

Subtype refines behavior depending on type.
### For Data Block Descriptors:
| ST  | Meaning                           |
| --- | --------------------------------- |
| 0x0 | Address = system physical address |
### For Keyed Data Block:
|ST|Meaning|
|---|---|
|0x0|RDMA / keyed memory usage|

# Type-specific field (bits 15:0)

Usage depends on descriptor type:
### Data Block / Segment
- Typically **Reserved (0)**

### Keyed Data Block
- Contains:
    - **Memory Key**
    - Used in NVMe-oF / RDMA

### Canonical Data Block descriptor
```
DW0–1: 64-bit Address
DW2:    Length
DW3:
  [31:24] Reserved
  [23:20] ST = 0
  [19:16] TD = 0 (Data Block)
  [15:0 ] Reserved
```


### Segment Descriptor (chaining)
```
DW0–1: Pointer to another SGL segment
DW2:    Length of that segment (in bytes)
DW3:
  ST = 0
  TD = 2 (Segment) or 3 (Last Segment)
```

---
# Key implementation nuances

### 1. Alignment

- Address alignment depends on:
    - Controller capability
    - Transport (PCIe vs NVMe-oF)

### 2. Chaining

- Segment descriptors allow:
    - Linked-list style traversal
    - Large or fragmented SGLs

### 3. Bit Bucket descriptor

- Used to **discard data** (rare but useful in testing)

---
Here is the **precise mapping of an SGL descriptor into an NVMe Submission Queue Entry (SQE)**, at the DWORD level, per the NVMe Base spec.

# 1) NVMe SQE (64 bytes) — where SGL lives

An NVMe command is **64 bytes = 16 DWORDs (DW0–DW15)**.
```
DW0   : OPC | FUSE | CID
DW1   : NSID
DW2–3 : Reserved / Metadata Ptr (MPTR) depending on command
DW4–5 : MPTR (Metadata Pointer)
DW6–7 : DPTR (Data Pointer)  <── PRP or SGL goes here
DW8–15: Command-specific fields
```

👉 The **Data Pointer (DPTR)** field is where **PRP or SGL is encoded**.


# 2) Switching from PRP → SGL

Whether DW6–DW7 contains PRPs or SGL is determined by:

- **PSDT (PRP or SGL Data Transfer)** field in **DW0 bits [15:14]**
```
|PSDT|Meaning|
|---|---|
|00b|PRP|
|01b|SGL (single descriptor)|
|10b|SGL (last segment descriptor)|
```


# 3) Exact embedding of SGL in SQE (DW6–DW9)

When SGL is used, the **first SGL descriptor is placed inline** in the SQE:
```
# 3) Exact embedding of SGL in SQE (DW6–DW9)

When SGL is used, the **first SGL descriptor is placed inline** in the SQE:
```

So:

> **DW6–DW9 of the SQE = one complete 16-byte SGL descriptor**


# 4) Bit-accurate mapping DW6–DW7 (Address)

```
DW6: bits 31:0   → Address[31:0]
DW7: bits 63:32  → Address[63:32]
```

## DW8 (Length)
```
DW8: bits 31:0 → Length (bytes)
```

## DW9 (Descriptor Control)
```
DW9 layout:

Bits 31:24 → Reserved
Bits 23:20 → Subtype (ST)
Bits 19:16 → Type (TD)
Bits 15:0  → Type-specific (e.g., key)
```

# 5) Example: Simple Data Block SGL in SQE

Assume:
- Buffer address = `0x00000012_34567000`
- Length = `0x2000` (8 KB)
- Descriptor type = Data Block (TD = 0)

```
DW6 = 0x34567000
DW7 = 0x00000012
DW8 = 0x00002000
DW9 = 0x00000000   (ST=0, TD=0)
```


# 6) Chained SGL (Segment Descriptor case)

If the transfer requires multiple descriptors:

- The inline SGL (DW6–DW9) becomes a **Segment Descriptor**

```
DW6–DW7 → pointer to SGL segment in host memory
DW8     → size of that segment
DW9     → TD = 2 (Segment) or 3 (Last Segment)
```

Then:

- Controller fetches additional SGL descriptors via DMA


# 7) Relationship to command execution

Flow:
1. Host submits SQE with:
    - PSDT = SGL
    - Inline descriptor in DW6–DW9
2. Controller:
    - Reads inline SGL descriptor
    - If Data Block → directly performs DMA
    - If Segment → walks SGL chain


# 8) PRP vs SGL in SQE (side-by-side)

|Field|PRP mode|SGL mode|
|---|---|---|
|DW6|PRP1|SGL Address[31:0]|
|DW7|PRP2|SGL Address[63:32]|
|DW8|(part of PRP list)|SGL Length|
|DW9|—|SGL Type/Control|

# 9) Important constraints

- Inline SGL descriptor must be **16-byte aligned**
- Controller capability:
    - Must advertise **SGL support (Identify Controller)**
- Not all commands support SGL (depends on spec revision and controller)


---
# Mental model

Each SGL entry says:

> “Here’s a buffer (or another list), here’s how big it is, and here’s how to interpret it.”

---

# Key takeaway

> An NVMe SGL descriptor is a **16-byte structure (DW0–DW3)** consisting of a **64-bit address, 32-bit length, and a 32-bit control field encoding descriptor type and subtype**, enabling flexible, chained descriptions of host memory for DMA

If you want, I can go deeper into:

- Exact **SGL descriptor bit layout (DW0–DW3)**
- How SGL is encoded in **NVMe submission queue entries**
- Or compare **PRP list vs SGL chain traversal at the controller level**

---
# Submission Queue EXAMPLE of using SGL descriptor

# 🔹 Scenario

- Command: **NVM Write**
- Data buffer is **non-contiguous**, so SGL is used
- Two segments:
    - Segment A → 4 KB @ `0x00000010_00000000`
    - Segment B → 8 KB @ `0x00000020_00000000`

We’ll use:

- **Data Block descriptor** for simple case
- Single descriptor inline (pointing to a segment list)

# 🔹 SQE Layout (Relevant Part)

```
DW6–DW9 → SGL Descriptor (16 bytes)
```

# 🔹 Example SQE (DWORD Dump)

```
DW0  : 0x00010001   (OPC=0x01 Write, CID=1)
DW1  : 0x00000001   (NSID = 1)

DW2–DW5 : 0x00000000

DW6  : 0x00000000   (SGL Address low)
DW7  : 0x00000030   (SGL Address high → 0x00000030_00000000)

DW8  : 0x00000020   (Length = 32 bytes → size of SGL list)

DW9  : 0x00000002   (Type = Segment Descriptor)

DW10–DW15 : command-specific (LBA, length, etc.)
```

# 🔹 Decode of SGL Descriptor (DW6–DW9)

## ✔ Address (DW6–DW7)

```
0x00000030_00000000
```

👉 Points to **SGL segment list in memory**

## ✔ Length (DW8)

```
0x20 (32 bytes)
```

👉 Size of SGL list (2 descriptors × 16 bytes each)

## ✔ Type (DW9)

```
0x02 → Segment Descriptor
```

👉 Means:

- “Go to this address and read a list of descriptors”

# 🔹 SGL Segment List in Host Memory

At address `0x00000030_00000000`:

## Descriptor 1 (16 bytes)

```
Address : 0x00000010_00000000
Length  : 0x00001000 (4 KB)
Type    : Data Block
```

## Descriptor 2 (16 bytes)

```
Address : 0x00000020_00000000
Length  : 0x00002000 (8 KB)
Type    : Data Block
```

# 🔹 Visual Flow

```
SQE
 ↓
DW6–9 → SGL Segment Descriptor
 ↓
Points to SGL List
 ↓
[Data Block A] → 4 KB
[Data Block B] → 8 KB
```

# 🔹 Key Variants

## ✔ Inline Data Block Descriptor (no segment list)

If only one segment:

```
DW6–7 → data address
DW8   → length
DW9   → Data Block Descriptor type
```

👉 No extra memory indirection

## ✔ Last Segment Descriptor

- Marks final segment list
- Prevents further chaining

# 🔹 Important Notes

- SGL descriptor = **always 16 bytes**
- DW9 encodes:
    - Descriptor Type
    - Subtype
- Address alignment rules depend on type

# 🔹 Key Insight

> The SQE doesn’t hold all buffer addresses—it holds a **pointer to a structured description of memory segments**.

---

# 🔹 One-Line Summary

**In an NVMe SQE, an SGL descriptor (DW6–DW9) either directly describes a data buffer or points to a list of descriptors in memory that collectively define a scattered I/O buffer.**