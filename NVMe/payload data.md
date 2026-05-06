

> In NVMe, **payload data** is the **actual user data being transferred** between the host and the SSD as part of an I/O command. It’s the _content_ (bytes) you care about—not the command metadata.

---

# 🔹 Where Payload Data Fits

```
Submission Queue Entry (SQE) → command + pointers
Completion Queue Entry (CQE) → status/result
Payload Data               → actual data in host memory
```

- **SQE** carries _what to do_ (read/write, LBA, length)
- **CQE** carries _result/status_
- **Payload** carries _the data itself_

# 🔹 Where Payload Lives

Payload data resides in a host memory [[data buffer]] and is accessed by the controller via **DMA** using:

- **PRP (Physical Region Page) pointers**, or
- **SGL (Scatter-Gather List) descriptors**

```
PRP/SGL → points to → payload buffer in host RAM
```

# 🔹 Read vs Write (Direction of Payload)

## ✔ Write (Host → SSD)

```
Payload = data in host buffer
SSD reads it via DMA → stores in NAND
```

## ✔ Read (SSD → Host)

```
Payload = data fetched from NAND
SSD writes it via DMA → into host buffer
```

# 🔹 Example

### Write 4 KB at LBA 100

```
Payload buffer (host RAM):
  Address = 0x10000000
  Size    = 4 KB

SQE:
  OPC = Write
  PRP1 = 0x10000000
```

👉 That 4 KB region is the **payload data**

# 🔹 What Is NOT Payload

|Component|Is it payload?|
|---|---|
|SQE (commands)|❌|
|CQE (status)|❌|
|Metadata (PI, etc.)|❌ (separate)|
|**User data buffer**|✅|

# 🔹 Payload vs Metadata

NVMe can optionally include **metadata** alongside payload:

```
Payload   → actual user data (e.g., 4 KB block)
Metadata  → extra info (e.g., protection info, checksum)
```

Metadata may be transferred via:

- Separate buffer (MPTR)
- Interleaved (extended LBA)

# 🔹 Size of Payload

Defined by:

```
Number of Logical Blocks (NLB) × LBA size
```

Example:

- LBA size = 4 KB
- NLB = 7 → payload = 8 × 4 KB = 32 KB

---
# Map payload into a real NVMe trace (hex + buffer)

> Small synthetic trace and readable mapping: one 4 KiB write with PRP1 pointing to a host buffer, then will show how the SQE fields identify the payload.

Thought for a couple of seconds

Below is a **realistic NVMe Write SQE + host buffer mapping**.

## 1. Assumptions

```
Command      : NVM Write
NSID         : 1
CID          : 0x0042
Starting LBA : 0x0000000000000100
LBA size     : 4096 bytes
Transfer     : 1 logical block = 4 KiB
Host buffer  : physical address 0x00000010_00002000
```


## 2. NVMe Write SQE hex dump

64-byte [[Submission Queue Entry (SQE)]]:

```
Offset  Bytes
00:     01 00 42 00
04:     01 00 00 00
08:     00 00 00 00
0C:     00 00 00 00
10:     00 00 00 00
14:     00 00 00 00
18:     00 20 00 00
1C:     10 00 00 00
20:     00 00 00 00
24:     00 00 00 00
28:     00 01 00 00
2C:     00 00 00 00
30:     00 00 00 00
34:     00 00 00 00
38:     00 00 00 00
3C:     00 00 00 00
```

## 3. SQE decoded by DWORD

```
DW0  = 0x00420001
       OPC = 0x01  → Write
       CID = 0x0042

DW1  = 0x00000001
       NSID = 1

DW4–DW5 = 0x0000000000000000
       MPTR unused

DW6–DW7 = 0x0000001000002000
       PRP1 = host data buffer address

DW8–DW9 = 0x0000000000000000
       PRP2 unused because transfer fits in one 4 KiB page

DW10–DW11 = 0x0000000000000100
       Starting LBA = 0x100

DW12 = 0x00000000
       NLB = 0 → transfer 1 logical block
```

Important: in NVMe NVM Read/Write, **[[Number of Logical Blocks (NLB)]] is zero-based**.
```
NLB = 0  → 1 block
NLB = 1  → 2 blocks
NLB = 7  → 8 blocks
```

The actual payload is **not inside the SQE**. It lives at:
```
PRP1 = 0x00000010_00002000
```

Example 4 KiB host buffer content:
```
Physical address 0x00000010_00002000

0000: 48 65 6C 6C 6F 20 4E 56  4D 65 20 50 61 79 6C 6F
0010: 61 64 20 44 61 74 61 21  00 00 00 00 00 00 00 00
0020: AA AA AA AA AA AA AA AA  AA AA AA AA AA AA AA AA
0030: BB BB BB BB BB BB BB BB  BB BB BB BB BB BB BB BB
...
0FF0: FF FF FF FF FF FF FF FF  FF FF FF FF FF FF FF FF
```

ASCII view of first bytes:
```
48 65 6C 6C 6F 20 4E 56 4D 65 ...
 H  e  l  l  o     N  V  M  e
```

So the command means:
```
Write 4096 bytes from host physical address 0x00000010_00002000
to namespace 1, starting at LBA 0x100.
```

## 5. End-to-end mapping

```
SQE
 ├─ OPC  = Write
 ├─ NSID = 1
 ├─ SLBA = 0x100
 ├─ NLB  = 0 → 1 block
 └─ PRP1 = 0x00000010_00002000
              ↓
        Host payload buffer
              ↓ DMA read by SSD
        NAND write via FTL
```

The **payload** is the 4 KiB buffer. The SQE only tells the controller **where that payload is** and **where to write it on the SSD**.

---
# 🔹 Key Insight

> NVMe queues move **commands**, but the **payload data moves via DMA directly between SSD and host memory**.

---
# 🔹 One-Line Summary

**Payload data in NVMe is the actual user data being transferred during read/write operations, stored in host memory and accessed by the SSD via PRP or SGL pointers.**

If you want, I can next:

- show **payload + metadata transfer layout (PI, DIF/DIX)**
