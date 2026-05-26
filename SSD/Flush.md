
> **Flush in NVMe** is an I/O command that forces the SSD to **persist all previously written data to non-volatile media** (e.g., NAND) **before reporting completion**. It establishes a **durability barrier**.

# 🔹 What Flush Guarantees

```
Before FLUSH completes:  All prior writes → safely stored in non-volatile media
```

👉 It prevents data being left only in **volatile caches** (e.g., DRAM).

# 🔹 Command Basics

- **Command**: Flush
- **Opcode**: `0x00` (NVM I/O command set)
- **Queue**: I/O Submission Queue
- **Data transfer**: none (no PRP/SGL payload)

# 🔹 How It Works Internally

Typical controller sequence:

```
1) Drain write pipeline
2) Move/commit any cached data
3) Update mapping metadata (FTL)
4) Ensure NAND program completion
5) Acknowledge completion
```

If the device has **power-loss protection (PLP)** capacitors, it may already guarantee persistence on power loss; Flush still enforces an **ordering point**.

# 🔹 Flush vs Write Completion

- A normal **Write** may complete when data is:
    - in controller buffers, or
    - scheduled for NAND
- **Flush** ensures:
    - data is **actually durable** in NAND (or equivalently protected)

# 🔹 Flush vs FUA (Force Unit Access)

|Feature|Scope|Effect|
|---|---|---|
|**Flush**|Global (all prior writes)|Creates a durability barrier|
|**FUA bit (in Write)**|Per command|That write bypasses/forces cache to be durable before completion|

👉 Use FUA for per-write durability; use Flush to **order and persist a batch**.

# 🔹 Ordering Semantics

```
Write A
Write B
Flush
Write C
```

Guarantee:

```
After Flush completes:
  A and B are durable
C may or may not be durable
```


# 🔹 Performance Considerations

- Flush can add **[[latency]]** (forces pipeline drain)
- Frequent flushes reduce throughput
- Filesystems/databases batch writes to minimize flush frequency

# 🔹 When Flush Is Used

- Filesystem sync operations (fsync)
- Databases (WAL commit)
- Journaling boundaries
- VM disk barriers

# 🔹 Edge Cases / Notes

- On devices with strong PLP, flush may be fast but still enforces **ordering**
- If **volatile write cache** is disabled, writes may already be durable—Flush still provides a **barrier**
- Applies per namespace (NSID in command)

# 🔹 Minimal SQE View

```
DW0  : OPC=0x00 (Flush)
DW1  : NSID
DW10–DW15 : unused
PRP/SGL : not used
```

---

# 🔹 One-Line Summary

**NVMe Flush is an I/O command that ensures all prior writes are durably committed to non-volatile media, providing a global persistence and ordering barrier.**
