

## Namespace Data Structure

In NVMe, the **Namespace Data Structure** is the **Identify Namespace** payload returned by the Identify command when:

```
CNS = 0x00
```

> It is a **4096-byte structure** that describes the **geometry, capacity, format, and capabilities of a namespace** (i.e., a logical block device).



## 🧠 What a Namespace Represents

A **namespace** is the unit the host actually uses for I/O:

```
Namespace = logical storage volume (like /dev/nvme0n1)
```

The Identify Namespace structure tells the host **how to access it correctly**.


## 📦 High-Level Layout (Key Regions)

```
0x000–0x03F   Core capacity & format
0x040–0x07F   Features & capabilities
0x068–0x07F   GUIDs (NGUID, EUI-64)
0x080–0x0FFF  LBA formats, metadata, vendor-specific
```

## 🔑 Critical Fields (Must Know)

### 🔹 1. NSZE, NCAP, NUSE — Capacity

```
Bytes 0–7   : NSZE (Namespace Size)
Bytes 8–15  : NCAP (Namespace Capacity)
Bytes 16–23 : NUSE (Namespace Utilization)
```

- **NSZE** → total logical blocks
- **NCAP** → usable blocks
- **NUSE** → currently used

👉 Host computes size as:

```
Size (bytes) = NSZE × LBA size
```


### 🔹 2. LBA Format (LBAF)

```
Byte 26 : NLBAF (number of LBA formats)
Byte 27 : FLBAS (current format)
```

Each LBA format entry defines:

```
LBA size = 2^ds
Metadata size = ms
```

Example:

```
ds = 9  → 512 bytes
ds = 12 → 4096 bytes
```




### 🔹 3. Metadata Capabilities

```
Byte 24 : MC (Metadata Capabilities)
Byte 25 : DPC (Data Protection Capabilities)
Byte 30 : DPS (Data Protection Settings)
```

- Indicates support for:
    - **T10 DIF / Protection Information**
    - Metadata location (separate vs inline)

### 🔹 4. NGUID & EUI-64 (Global IDs)

```
Bytes 104–119 : NGUID (128-bit)
Bytes 120–127 : EUI-64 (64-bit)
```

- Used for **persistent global identification**
- Critical in multipath / NVMe-oF

### 🔹 5. NVM Capacity (Advanced)

```
Bytes 128–135 : NVM Capacity
```

- May differ from NSZE in thin provisioning scenarios


### 🔹 6. ANA / Multipath (if supported)

Fields indicate:

- Asymmetric Namespace Access states
- Path optimization info


### 🔹 7. Endurance Group / NVM Set

```
Byte ~ 516+ (NVMe 1.4+)
```

- Maps namespace to:
    - Endurance group
    - NVM set


## 📊 LBA Format Table (Important Section)

Each entry (~4 bytes):

```
Bits:
[7:0]   → Metadata size (ms)
[15:8]  → LBA data size exponent (ds)
[17:16] → Relative performance (rp)
```


## 🔧 Practical Example

Using **nvme-cli**:

```
nvme id-ns /dev/nvme0n1
```

Output:

```
nsze    : 3750748848
ncap    : 3750748848
nuse    : 1200000000
nlbaf   : 2
flbas   : 0
lbaf 0  : ms:0   ds:9   → 512B
lbaf 1  : ms:0   ds:12  → 4K
nguid   : 601c3a927f4b11ee9a3c0242ac120002
eui64   : 0025385a01438abc
```

## ⚙️ How It’s Used by the Host

The OS/driver uses this structure to:

- Determine **sector size**
- Calculate **device capacity**
- Configure **I/O alignment**
- Enable **data protection (PI)**
- Identify device across paths

---

## ⚠️ Common Pitfalls

- Misinterpreting **NSZE vs NCAP**
- Ignoring LBA format → misaligned I/O
- Not checking NGUID/EUI64 → multipath issues
- Overlooking metadata → corruption in PI-enabled systems

## 🧩 Key Insight

> **The Identify Namespace structure is the “contract” between host and SSD—it defines exactly how data is addressed, formatted, and protected.**

---

If you want, I can:

- Give **full byte-by-byte (DW0–DW255) breakdown**
- Map fields to **actual NVMe command behavior**
- Explain how it interacts with **CSI (ZNS, KV, etc.)**