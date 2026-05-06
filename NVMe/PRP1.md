
> In NVMe, **PRP1** and **PRP2** are the two data pointer fields inside a Submission Queue Entry (SQE) used to describe where the data buffer resides in host memory. The key difference is **how they’re used depending on transfer size**.

# 🔹 1) What PRP1 and PRP2 Are

- **PRP = Physical Region Page**
- Each PRP is a **64-bit physical address**

Located in SQE:

```
DW6–DW7 → PRP1
DW8–DW9 → PRP2
```

# 🔹 2) Core Difference

| Field    | Role                                                                              |
| -------- | --------------------------------------------------------------------------------- |
| **PRP1** | Always points to the **first memory page** of the data buffer                     |
| **PRP2** | Either points to the **second page** OR a **PRP list** (for multi-page transfers) |
# 🔹 3) How They Are Used (Critical)

## ✔ Case 1: Data fits in ONE page

```
[ PRP1 → full buffer ]
PRP2 = unused
```

## ✔ Case 2: Data spans TWO pages

```
PRP1 → first page
PRP2 → second page
```

## ✔ Case 3: Data spans MORE than two pages

```
PRP1 → first page
PRP2 → pointer to PRP List
```

### PRP List:

A contiguous array of physical page addresses:

```
PRP List:
[ Page2 ][ Page3 ][ Page4 ] ...
```


# 🔹 4) Visual Summary

```
Case 1 (≤1 page):
  PRP1 → Data
  PRP2 → not used

Case 2 (=2 pages):
  PRP1 → Page 1
  PRP2 → Page 2

Case 3 (>2 pages):
  PRP1 → Page 1
  PRP2 → PRP List → Page 2,3,4,...
```

# 🔹 5) Alignment Rules (Important)

- PRP addresses must be **page-aligned** (except first page offset allowed)
- Controller assumes:
    - Page size = system memory page (commonly 4KB)

# 🔹 6) Why This Design Exists

- Keeps SQE size fixed (64 bytes)
- Efficient for:
    - Small I/O (fast path via PRP1)
    - Large I/O (scalable via PRP list)

# 🔹 7) Relation to [[Scatter-Gather List (SGL)]]

- PRP = simpler, page-based
- SGL = more flexible (non-contiguous buffers)

👉 Modern NVMe may use SGL instead of PRP for complex mappings

# 🔹 8) Key Insight

> **PRP1 always points to data.  
> PRP2 conditionally points to either more data or a list of pointers.**

# 🔹 One-Line Summary

**PRP1 holds the first data page address, while PRP2 holds either the second page address or a pointer to a PRP list when the data spans more than two pages.**

---

If you want, I can:

- show exact **PRP list binary layout**
- or map PRP usage into a **real NVMe read/write command example**