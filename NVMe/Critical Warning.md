
> The **Critical Warning** field in the NVMe SMART log (Log Page `0x02`) is a **bitmask (1 byte)** that flags serious health or reliability conditions. Each bit represents a specific issue.
# Bit-Level Definition

```
Bit 0 → Available Spare below threshold
Bit 1 → Temperature exceeded
Bit 2 → Reliability degraded
Bit 3 → Media placed in read-only mode
Bit 4 → Volatile memory backup failed
Bit 5–7 → Reserved
```


## Bit 0 — Available Spare Below Threshold

```
0x01
```

- Spare NAND blocks are running low
- SSD is nearing wear-out

👉 Action:

- Monitor closely
- Plan replacement

## ✔ Bit 1 — Temperature Threshold Exceeded

```
0x02
```

- SSD temperature exceeded safe limit
- May trigger thermal throttling or shutdown

👉 Risks:

- Performance degradation
- Long-term damage

## ✔ Bit 2 — Reliability Degraded

```
0x04
```

- General reliability issue detected
- Could be due to:
    - excessive media errors
    - internal failures

👉 Serious condition → investigate immediately

## ✔ Bit 3 — Media in Read-Only Mode

```
0x08
```

- SSD has switched to **read-only**
- Writes are no longer allowed

👉 Typically caused by:

- Endurance exhaustion
- Critical internal failure

## ✔ Bit 4 — Volatile Memory Backup Failed

```
0x10
```

- Failure in **power-loss protection (PLP)** system
- SSD cannot safely flush DRAM on power loss

👉 Risk:

- Data loss during power failure

## ✔ Bits 5–7 — Reserved

- Not currently defined

# 🔹 Example Decoding

## Example 1:

```
critical_warning = 0x01
```

→ Only Bit 0 set  
→ Spare capacity below threshold

## Example 2:

```
critical_warning = 0x0A
```

```
0x0A = 00001010 (binary)
        ↑   ↑
       bit3 bit1
```

→ Media read-only + temperature exceeded

## Example 3:

```
critical_warning = 0x1F
```

→ Bits 0–4 all set  
→ Severe failure condition

# 🔹 Severity Ranking

|Bit|Severity|
|---|---|
|Bit 0|Warning|
|Bit 1|Warning / thermal|
|Bit 2|High|
|Bit 3|Critical (writes disabled)|
|Bit 4|Critical (data safety risk)|
# 🔹 Key Insight

> Critical Warning is a **quick health summary bitmask**—it tells you immediately if the drive is in trouble without parsing all SMART fields.

---

# 🔹 One-Line Summary

**The NVMe Critical Warning field is a bitmask that flags major health issues like low spare capacity, overheating, degraded reliability, read-only mode, and PLP failure.**

---

If you want, I can:

- correlate each bit with **SMART metrics (e.g., percentage_used, temperature)**
- or show **real failure scenarios and how these bits evolve over time**
