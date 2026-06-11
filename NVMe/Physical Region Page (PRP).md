
> A **Physical Region Page (PRP)** in NVMe is the **primary mechanism used by the host to tell the controller where data buffers reside in system memory for DMA transfers**.

### Core idea

NVMe commands (read/write) need to move data between:

- Host memory (DRAM)
- NVMe controller (SSD)

Instead of copying data, NVMe uses **DMA (Direct Memory Access)**.  
PRPs provide the **physical memory addresses** that the controller should read from or write to.

> **PRP = a pointer (or list of pointers) to physical memory pages used for I/O**

### Where PRP appears

PRPs are embedded in the **[[Submission Queue]] Entry (SQE)** of an NVMe command:

- **[[Physical Region Page 1 (PRP1)]]** → first data buffer (always present)
- **[[PRP2]]** → either:
    - second buffer, OR
    - pointer to a **PRP List**

### PRP structure and usage

1. Case 1

- Points to the **first memory page**
- Can start at any offset within that page

 2. Case 2
	 #### Case A: Small transfer (fits in 2 pages)
	- PRP2 directly points to the second page

	Case B: Larger transfer (> 2 pages)
	- PRP2 points to a **PRP List**

---

### PRP List

A **PRP list** is an array of physical page addresses:
```
PRP List:
[ addr_page_2 ]
[ addr_page_3 ]
[ addr_page_4 ]
...
```

Each entry:

- Points to a **full memory page**
- Pages must be **physically contiguous within each page**, but not across pages

---
### Example

Suppose a 16 KB read with 4 KB page size:
```
Total = 4 pages

PRP1 → page 1
PRP2 → PRP list

PRP list:
  → page 2
  → page 3
  → page 4
```

---
### Key constraints

- Memory must be:
    - **Physically addressable (DMA-capable)**
    - **Page-aligned** (except first PRP which can have offset)
- Page size is defined by:
    - Controller capability (CC.MPS)

---

### Physical Region Page (PRP) vs [[Scatter-Gather List (SGL)]] (quick comparison)

| Feature      | PRP                   | SGL                       |
| ------------ | --------------------- | ------------------------- |
| Structure    | Page-based            | Descriptor-based          |
| Flexibility  | Limited               | High                      |
| Complexity   | Low                   | Higher                    |
| Common usage | Standard NVMe SSD I/O | NVMe-oF, advanced drivers |

### Why PRP is efficient

- Minimal overhead (just addresses)
- Optimized for:
    - Page-based memory systems
    - High-throughput sequential I/O
- Simple for hardware to traverse

### Mental model

Think of PRP like:

> “Here are the exact physical pages in RAM—go read/write data directly from/to them.”

---

If you want, I can go deeper into:

- Exact **PRP alignment rules and edge cases**
- How PRP interacts with **IOMMU / DMA mapping**
- Or a **DW-level breakdown of PRP fields inside an SQE**
