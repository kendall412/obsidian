
> **Atomicity in NVMe** means that a write is **all-or-nothing**: after a failure (e.g., power loss), the data for that write is seen as **either fully old or fully new—never partially updated**.

# 🔹 Why Atomicity Matters

Without atomicity:

```
Power loss during write →
Some sectors updated, others not →
Data corruption / torn write
```

With atomicity:

```
After failure →
Either entire write is committed
OR nothing is changed
```

# 🔹 Where NVMe Guarantees Atomicity

## ✔ Atomic Write Unit (AWU)

NVMe defines a size (in logical blocks) that is **guaranteed atomic**:

- **AWUN** → Atomic Write Unit Normal
- **AWUPF** → Atomic Write Unit Power Fail

These are reported in **Identify Controller / Namespace data**.

## ✔ Interpretation

```
If write size ≤ AWU:
  → guaranteed atomic

If write size > AWU:
  → may be torn on failure
```

# 🔹 Example

Assume:

- LBA size = 4 KB
- AWUPF = 7 (zero-based → 8 blocks)

```
Atomic size = 8 × 4 KB = 32 KB
```

### Cases:

```
Write 16 KB → atomic (safe)
Write 32 KB → atomic (safe)
Write 64 KB → NOT guaranteed atomic
```

# 🔹 How SSD Ensures Atomicity

Internally via FTL:

- Write data to new pages
- Update mapping **only after full write completes**
- Use:
    - journaling
    - metadata protection
    - capacitor-backed flush (PLP)

# 🔹 Atomicity vs Power Loss Protection (PLP)

|Feature|Role|
|---|---|
|Atomicity|Logical guarantee (no torn writes)|
|PLP (capacitors)|Ensures data reaches NAND during power loss|

👉 PLP helps enforce atomicity in real hardware.

# 🔹 Related NVMe Features

## ✔ Write Atomicity Normal (Feature ID 0x0A)

- Controls atomic write behavior

## ✔ FUA (Force Unit Access)

- Ensures data is persisted before completion

## ✔ Flush command

- Forces durability boundary

# 🔹 Atomicity vs Consistency

- Atomicity ensures **single write correctness**
- Does NOT ensure:
    - multi-write transaction consistency
    - filesystem integrity

👉 Higher-level software (DB, FS) handles that

# 🔹 Real-World Use

- Databases (avoid torn pages)
- Journaling filesystems
- WAL (write-ahead logging)

---
# FTL journaling enforces atomicity step-by-step

FTL journaling enforces atomicity by making the **mapping update** atomic, not necessarily by overwriting NAND data in place.

Flash cannot overwrite in place, so the SSD uses a **copy-on-write + journal/metadata commit** model.
## Example: host writes LBA 100

Initial state:

```
Mapping table:
LBA 100 → Physical Page A

Page A contains old data
```

Host sends new data for LBA 100.

## Step-by-step atomic write flow

### 1. Allocate a new free physical page

```
Old mapping:
LBA 100 → Page A

New page allocated:
Page B
```

The SSD does **not** overwrite Page A.

### 2. Write new data to Page B

```
Page A = old data
Page B = new data, not yet committed
```

At this point, if power fails, the SSD can still recover using Page A.

### 3. Write journal record

The FTL writes a small metadata record like:

```
Journal entry:
  LBA = 100
  old physical page = A
  new physical page = B
  sequence number
  checksum / CRC
  commit marker = not committed
```

This journal record says:

> “I am about to change LBA 100 from Page A to Page B.”

### 4. Commit the journal entry

After the new data and journal metadata are safely written, the SSD marks the journal entry as committed:

```
Journal entry:
  LBA 100: A → B
  commit marker = valid
```

This commit marker is the atomic boundary.

### 5. Update the active mapping table

Now the FTL updates its logical-to-physical map:

```
Before:
LBA 100 → Page A

After:
LBA 100 → Page B
```

### 6. Mark old page invalid

```
Page A = invalid / stale
Page B = current valid data
```

Page A will later be reclaimed during garbage collection.

## What happens during power loss?

### Case A: Power fails before journal commit

```
Page B may exist
Journal not committed
Mapping still points to Page A
```

Recovery result:

```
LBA 100 → Page A
```

The host sees old data.

### Case B: Power fails after journal commit

```
Journal says A → B committed
Mapping update may or may not have reached flash
```

During recovery, the SSD replays the journal:

```
LBA 100 → Page B
```

The host sees new data.

## Atomicity result

After recovery, there are only two legal outcomes:

```
Old data: LBA 100 → Page A
```

or:

```
New data: LBA 100 → Page B
```

There is no valid state where LBA 100 points to half-old / half-new data.

## Multi-block write example

For a write covering several LBAs:

```
LBA 100 → Page B
LBA 101 → Page C
LBA 102 → Page D
LBA 103 → Page E
```

The journal records the whole group:

```
Transaction:
  100: A → B
  101: F → C
  102: G → D
  103: H → E
  checksum
  commit marker
```

Only after all new pages are written does the FTL commit the transaction.

If power fails before commit:

```
All LBAs remain old
```

If power fails after commit:

```
All LBAs become new
```

That is how the SSD prevents a **torn write** within its guaranteed atomic write size.

## Key point

FTL journaling does not make NAND writes magically indivisible. It makes the **logical mapping switch** indivisible:

```
Atomicity = old mapping OR new mapping
```

The host never sees an incomplete intermediate mapping state.

---
# 🔹 Key Insight

> Atomicity is guaranteed **only up to a certain size (AWU)**—beyond that, the host must manage consistency.

---

# 🔹 One-Line Summary

**Atomicity in NVMe ensures that writes up to a defined size (AWU) are completed fully or not at all, preventing partial or torn writes, especially during failures.**

---

If you want, I can:

- show **exact AWUN/AWUPF bit fields in Identify data**
