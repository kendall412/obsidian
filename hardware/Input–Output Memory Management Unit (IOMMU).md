
> **IOMMU (Input–Output Memory Management Unit)** is a hardware unit that sits between **devices (e.g., an NVMe SSD on PCIe)** and **system memory**, translating and controlling the addresses a device can use for [[Direct Memory Access (DMA)]].

# 🔹 What Problem It Solves

Devices doing DMA would otherwise use **physical addresses directly**, which is unsafe:

```
Device DMA → Physical memory (unrestricted)
```

👉 Risks:

- Corrupt arbitrary memory
- Read sensitive data
- No isolation between devices

# 🔹 What IOMMU Does

## ✔ Address Translation (like CPU MMU)

```
Device address (IOVA)
        ↓
IOMMU translation tables
        ↓
Physical memory address
```

👉 Device sees a **virtual I/O address (IOVA)**, not real RAM.

## ✔ Memory Protection

- Restricts device to specific memory regions
- Blocks illegal access

```
Allowed: 0x1000–0x2000Blocked: everything else
```

## ✔ Isolation

- Each device can have its **own address space**
- Prevents one device from interfering with another

# 🔹 IOMMU in NVMe (DMA Flow)

```
Host buffer (virtual)
        ↓
Driver maps buffer → IOVA
        ↓
SQE contains PRP/SGL = IOVA
        ↓
SSD issues DMA using IOVA
        ↓
IOMMU translates → physical address
```

👉 SSD never sees real physical memory directly.

# 🔹 Key Components

## ✔ IOVA (I/O Virtual Address)

- Address used by device

## ✔ Page Tables

- Managed by OS/driver
- Similar to CPU page tables

## ✔ Context per Device

- Each PCIe device can have its own mapping

# 🔹 Benefits

## ✔ Security

- Prevents malicious DMA attacks

## ✔ Virtualization

- Enables VM passthrough (each VM gets isolated device access)

## ✔ Flexibility

- Allows non-contiguous memory to appear contiguous to device

# 🔹 Example

Host allocates fragmented memory:
```
Physical:
  0xA000
  0xF000
```

IOMMU maps it as:
```
IOVA:
  0x1000 → 0xA000
  0x2000 → 0xF000
```

SSD sees:
```
0x1000, 0x2000 (contiguous logical space)
```

# 🔹 IOMMU vs MMU

|Feature|CPU MMU|IOMMU|
|---|---|---|
|Used by|CPU|Devices|
|Translates|Virtual → Physical|IOVA → Physical|
|Purpose|Process memory isolation|DMA isolation|
# 🔹 Real Implementations

- Intel: VT-d
- AMD: AMD-Vi
- ARM: SMMU

# 🔹 Trade-offs

## ✔ Pros

- Security
- Isolation
- Virtualization support

## ❌ Cons

- Added latency (translation overhead)
- Slight performance impact for DMA-heavy workloads

# 🔹 Key Insight

> IOMMU gives devices a **controlled, virtualized view of memory**, just like the CPU has via the MMU.

---

# 🔹 One-Line Summary

**IOMMU is a hardware unit that translates and restricts device DMA addresses, enabling secure and isolated access to system memory for devices like NVMe SSDs.**

---

If you want, I can show:

- how IOMMU interacts with **PRP/SGL mapping in Linux**
- or a **step-by-step DMA transaction including IOVA translation**