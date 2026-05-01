Here is the **exact, field-by-field breakdown of an NVMe Submission Queue Entry (SQE)** at the DWORD level (**DW0–DW15**, total 64 bytes), aligned with the NVMe Base specification.

# NVMe SQE Layout (64 bytes = 16 DWORDs)
```
DW0   : Opcode / Command Control
DW1   : Namespace Identifier (NSID)
DW2–3 : Metadata Pointer (MPTR)
DW4–5 : Data Pointer (DPTR) part 1
DW6–7 : Data Pointer (DPTR) part 2
DW8–15: Command-specific fields
```

---
# DW0 — Command Dword 0 (CDW0)
```
Bits  7:0   OPC   → Opcode
Bits  9:8   FUSE  → Fused operation
Bits 13:10  Reserved
Bits 15:14  PSDT  → PRP or SGL Data Transfer
Bits 31:16  CID   → Command Identifier
```

### Details

- **OPC**: Command type (e.g., Read = 0x02, Write = 0x01)
- **FUSE**:
    - 00 = normal
    - 01 = first fused command
    - 10 = second fused command
- **PSDT**:
    - 00 = PRP
    - 01 = SGL (single descriptor)
    - 10 = SGL (segment descriptor)
- **CID**:
    - Unique ID used to match completions
---
# DW1 — Namespace ID (NSID)
```
Bits 31:0 → NSID
```
- Identifies the target namespace
- Special values:
    - `0xFFFFFFFF` → all namespaces (admin commands)

---
# DW2–DW3 — Metadata Pointer (MPTR)
```
DW2: bits 31:0   → MPTR[31:0]
DW3: bits 63:32  → MPTR[63:32]
```

- Points to metadata buffer (if used)
- Otherwise set to 0

---
# DW4–DW5 and DW6–DW7 — Data Pointer (DPTR)

This 128-bit field is interpreted differently depending on **PSDT**.

### Case A: [[Physical Region Page (PRP)]] mode (PSDT = 00)
```
DW4–DW5 → PRP1 (first data buffer)
DW6–DW7 → PRP2 (second buffer or PRP list pointer)
```

### Case B: [[Scatter-Gather List (SGL)]] mode (PSDT = 01 or 10)
```
DW4–DW7 → 16-byte SGL descriptor (inline)
```

Mapping:
```
DW4 → Address[31:0]
DW5 → Address[63:32]
DW6 → Length
DW7 → Type/Subtype/Control
```

---
# DW8–DW15 — Command-Specific Fields

These depend on the opcode.

---
## Example: NVM Read/Write command

### DW10 — Starting LBA (SLBA low)
```
Bits 31:0 → SLBA[31:0]
```

### DW11 — Starting LBA (SLBA high)
```
Bits 63:32 → SLBA[63:32]
```

### DW12 — Length and Control
```
Bits 15:0  → NLB (Number of Logical Blocks, 0-based)
Bits 25:16 → Reserved / command-specific
Bits 31:26 → Control flags
```

### DW13 — Dataset Management / Hints
```
Bits 31:0 → Command-specific (e.g., access hints)
```

### DW14 — Protection / Metadata
```
Bits 31:0 → Protection Information / metadata fields
```

### DW15 — Reference Tag / App Tag
```
Bits 31:0 → End-to-end data protection fields
```

# Full SQE (annotated)
```
DW0   : OPC | FUSE | PSDT | CID
DW1   : NSID
DW2–3 : MPTR
DW4–5 : PRP1 / SGL Address
DW6–7 : PRP2 / SGL Length + Type
DW8–9 : Command-specific
DW10–11: SLBA (for read/write)
DW12  : NLB + control
DW13  : Hints
DW14  : Protection info
DW15  : Tags
```

---
# Key nuances

### 1. 0-based NLB

- `NLB = number_of_blocks - 1`
- Example: 1 block → NLB = 0

---

### 2. Command-specific variability

- Admin vs I/O commands use DW10–DW15 differently

---

### 3. Alignment requirements

- PRP and SGL addresses must be:
    - DMA-capable
    - Properly aligned

---

### 4. CID importance

- Used by host to track outstanding commands
- Returned in CQE

---

# Mental model

Think of the SQE as:

> A **fully self-contained command packet**:  
> “What to do (OPC), where (NSID + SLBA), how much (NLB), and where data lives (PRP/SGL).”

---

# Key takeaway

> An NVMe SQE is a **64-byte command structure (DW0–DW15)** combining **command identity, addressing, data pointers (PRP/SGL), and operation-specific parameters**, enabling fully asynchronous, high-performance I/O submission.

