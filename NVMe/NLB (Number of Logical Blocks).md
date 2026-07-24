
> **NLB (Number of Logical Blocks)** in NVMe is a field in read/write commands that specifies **how many logical blocks (LBAs) to transfer**—but it uses a **zero-based encoding**.

# 🔹 Where NLB Is Located

In an NVMe Read/Write Submission Queue Entry:
```
DW12:
  bits 15:0 → NLB (Number of Logical Blocks)
```

# 🔹 Key Rule (Very Important)

```
Actual blocks transferred = NLB + 1
```

# 🔹 Examples

|NLB value|Actual blocks|If LBA size = 4 KB|
|---|---|---|
|0|1 block|4 KB|
|1|2 blocks|8 KB|
|7|8 blocks|32 KB|
|255|256 blocks|1 MB|
# 🔹 Why Zero-Based?

NVMe uses zero-based encoding to:

- Maximize range with limited bits
- Align with hardware efficiency

# 🔹 How It Maps to Payload Size

```
Payload size = (NLB + 1) × LBA size
```

### Example

```
NLB = 3
LBA size = 4096 bytes

→ (3 + 1) × 4096 = 16 KB transfer
```

# 🔹 In a Real Command

```
DW10–DW11 → Starting LBA (SLBA)
DW12       → NLB
```

Meaning:
```
Transfer from SLBA
for (NLB + 1) consecutive logical blocks
```

# 🔹 Relation to PRP / SGL

NLB determines how much data is transferred, and:

- **PRP/SGL must describe enough buffer space**
- Controller uses:

```
Buffer size ≥ (NLB + 1) × LBA size
```

# 🔹 Common Mistake

❌ Assuming:

```
NLB = number of blocks
```

✅ Correct:

```
NLB = number of blocks - 1
```

# 🔹 Key Insight

> NLB defines the **transfer length in units of logical blocks**, not bytes, and always requires adding 1 to get the real count.

---

# 🔹 One-Line Summary

**NLB in NVMe is a zero-based field in read/write commands that specifies the number of logical blocks to transfer, where actual blocks = NLB + 1.**

---

If you want, I can:

- show how NLB interacts with **MDTS (max transfer size) limits**
- or decode NLB from a **real NVMe trace**



