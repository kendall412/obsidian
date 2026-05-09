### What is a **doorbell** in NVMe?

> A **doorbell** is a **memory-mapped I/O (MMIO) register** that the host writes to in order to **notify the NVMe controller that a queue pointer has been updated**.

Think of it as a lightweight hardware signal:
- **Submission Queue Tail Doorbell (SQT)**  
    Host writes → “I’ve added new commands up to this tail index—go fetch them.”
- **Completion Queue Head Doorbell (CQH)**  
    Host writes → “I’ve consumed completions up to this head index—space is free.”

No commands are carried in the doorbell write itself—only the **queue index**. The actual command data is already in host memory (the SQ).

### Where doorbells live: PCIe BAR mapping

NVMe exposes its controller registers via a PCIe **Base Address Register (BAR)**, typically **BAR0**. Within that MMIO space:
```
BAR0 base
  ├─ 0x0000 … 0x0FFF : Controller registers (CAP, VS, CC, CSTS, AQA, ASQ, ACQ, …)
  └─ 0x1000 …        : Doorbell register array
```

> **Doorbells start at offset `0x1000` from the BAR base.**
---
### Doorbell stride (CAP.DSTRD)

Doorbells are evenly spaced. The spacing is defined by the controller capability field **CAP.DSTRD**:
```
stride (bytes) = 2^(2 + DSTRD)
```

- DSTRD = 0 → 4 bytes
- DSTRD = 1 → 8 bytes
- DSTRD = 2 → 16 bytes
- …

### Exact address formulas

Let:

- `BAR` = mapped base of BAR0
- `DB_BASE = BAR + 0x1000`
- `stride = 2^(2 + DSTRD)`
- `QID` = queue identifier (0 = Admin queue)

### Submission Queue Tail Doorbell (SQT)
```
SQT(QID) = DB_BASE + (2 × QID) × stride
```

### Completion Queue Head Doorbell (CQH)
```
CQH(QID) = DB_BASE + (2 × QID + 1) × stride
```

---
### Layout pattern
```
DB_BASE (BAR + 0x1000)

QID=0:
  +0×stride → SQ0 Tail (Admin SQ)
  +1×stride → CQ0 Head (Admin CQ)

QID=1:
  +2×stride → SQ1 Tail
  +3×stride → CQ1 Head

QID=2:
  +4×stride → SQ2 Tail
  +5×stride → CQ2 Head
...
```

> Each **queue pair** (SQ/CQ) uses **two doorbells**.

---
### What the host writes

- **SQT(QID)**: write the **new tail index** of SQ `QID`
- **CQH(QID)**: write the **new head index** of CQ `QID`

Writes are typically **32-bit MMIO stores**.

### Ordering and performance notes

- **Ordering**: Ensure SQ entries are visible in memory _before_ ringing SQT  
    (use a write memory barrier in drivers).
- **Batching**: Hosts often batch multiple SQEs and ring once to reduce MMIO overhead.
- **No reads**: Doorbells are write-only from the host’s perspective.
---
### Minimal example

Assume:

- `BAR = 0x80000000`
- `DSTRD = 0` → `stride = 4`
- `QID = 1`

```
DB_BASE = 0x80001000

SQ1 Tail = 0x80001000 + (2×1)*4     = 0x80001008
CQ1 Head = 0x80001000 + (2×1 + 1)*4 = 0x8000100C
```

## Key takeaway

> A **doorbell** is an MMIO notification register. In NVMe, doorbells are mapped at **BAR + 0x1000**, spaced by **2^(2 + DSTRD)** bytes, with **two registers per queue (SQ tail and CQ head)** that the host writes to signal queue progress.

