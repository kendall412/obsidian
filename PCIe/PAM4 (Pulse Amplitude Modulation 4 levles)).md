# PAM4 (Pulse Amplitude Modulation with 4 levels) signaling

It is used in **PCIe 6.0** to double the amount of data sent per signal transition compared with older PCIe generations.

---

## Older PCIe signaling: NRZ

PCIe 1.0 through PCIe 5.0 use **NRZ** signaling.

NRZ has **2 voltage levels**:

```text
low  = 0
high = 1
```

So each symbol carries:

```text
1 bit
```

Example:

```text
0 1 1 0 1
```

---

## PCIe 6.0 signaling: PAM4

PCIe 6.0 uses **4 voltage levels**.

Each level represents **2 bits**:

| PAM4 level | Bits |
|---:|---:|
| Level 0 | 00 |
| Level 1 | 01 |
| Level 2 | 10 |
| Level 3 | 11 |

So each transmitted symbol carries:

```text
2 bits
```

Example:

```text
00 01 10 11
```

**Instead of sending one bit at a time, PAM4 sends two bits at a time**.

---

## Why PCIe uses PAM4

PCIe 5.0 runs at:

```text
32 GT/s
```

PCIe 6.0 runs at:

```text
64 GT/s
```

Rather than doubling the electrical frequency directly, PCIe 6.0 uses PAM4 so each signal transition carries twice as much information.

Conceptually:

```text
NRZ:  one symbol = 1 bit
PAM4: one symbol = 2 bits
```

This allows PCIe 6.0 to achieve higher data rate.

---

## Tradeoff of PAM4

The downside is that PAM4 has smaller voltage spacing between levels.

NRZ:

```text
0 -------------------- 1
```

PAM4:

```text
00 ------ 01 ------ 10 ------ 11
```

Because the levels are closer together, PAM4 is more sensitive to:

- noise
- signal distortion
- crosstalk
- jitter
- channel loss

So PAM4 links generally need stronger reliability features, such as:

- **FEC**, Forward Error Correction
- **FLIT mode**
- stronger equalization and link training

---

## Simple analogy

Imagine sending messages with a light:

### NRZ

```text
dim  = 0
bright = 1
```

One flash sends one bit.

### PAM4

```text
off       = 00
dim       = 01
medium    = 10
bright    = 11
```

One flash sends two bits.

This is faster, but it is harder to distinguish the levels accurately.

---

## Summary

**PAM4 in PCIe** means:

```text
4 signal levels instead of 2
2 bits per symbol instead of 1
used starting with PCIe 6.0
enables 64 GT/s signaling
requires FEC because it is more error-prone
```