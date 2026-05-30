
> In Computer Engineering, a **PCIe BAR** stands for **Base Address Register**. It is a register inside a PCIe device that tells the host system:

- **What type of memory/resource the device needs**
- **How large that resource is**
- **Where the CPU should map that resource in system address space**

Think of a BAR as the mechanism that lets the CPU access a PCIe device’s internal registers or onboard memory through normal memory reads/writes

# High-Level Concept

When a PCIe device powers up:

1. The device exposes several BAR registers in its PCIe configuration space.
2. The BIOS/UEFI or operating system enumerates the device.
3. The system determines how much address space the device needs.
4. The OS assigns physical addresses to the BARs.
5. The CPU can now communicate with the device using MMIO (Memory-Mapped I/O).

# Why BARs Exist

A PCIe device may contain:

- Control registers
- DMA engines
- Queue memory
- Firmware memory
- Doorbells
- Onboard RAM

The CPU needs a way to access these resources.

BARs provide that mapping.

# Example: NVMe SSD

An Non-Volatile Memory Express SSD typically exposes:

- Controller registers
- Admin queue registers
- I/O queue doorbells
- Interrupt configuration

through a PCIe BAR.

For example:

| Resource        | Accessed Through |
| --------------- | ---------------- |
| CAP register    | BAR0             |
| CC register     | BAR0             |
| CSTS register   | BAR0             |
| SQ/CQ doorbells | BAR0             |

The OS maps BAR0 into system memory space, so software can do (in C):

```
mmio_write32(bar0 + 0x1000, value);
```

to ring an NVMe doorbell.

# Where BARs Live

BARs are located in the device’s:

- PCIe Configuration Space
- Type 0 Configuration Header

Traditional PCI devices can have up to:

- 6 BARs (BAR0–BAR5)

# BAR Types

## 1. Memory BAR

Most common.

Maps device memory/registers into CPU memory address space.

CPU accesses with normal load/store instructions.

Example (in C):

```
*(volatile uint32_t *)(bar0 + offset) = value;
```


## 2. I/O BAR

Older mechanism using x86 I/O ports.

Accessed using instructions like (in asm):

```
in
out
```

Rare in modern PCIe devices.

NVMe does NOT use I/O BARs.

# 32-bit vs 64-bit BAR

## 32-bit BAR

Can map below 4 GB address space.

## 64-bit BAR

Can map above 4 GB.

High-performance devices usually use 64-bit BARs.

Modern GPUs and NVMe SSDs commonly use 64-bit BARs.

# BAR Size Detection

The OS determines BAR size using a special procedure.

Example:

1. Save original BAR value
2. Write all 1s:

```
0xFFFFFFFF
```

3. Read back value
4. Device hardwires size bits to 0
5. OS calculates required size

Example:

```
Read back: 0xFFFFF000
```

Means:

```
Size = 0x1000 bytes = 4 KB
```

# BAR Layout Example

Example NVMe BAR0:

| Offset | Register  |
| ------ | --------- |
| 0x0000 | CAP       |
| 0x0008 | VS        |
| 0x0014 | CC        |
| 0x001C | CSTS      |
| 0x1000 | Doorbells |

The BAR base address might be:

```
0x80000000
```

So:

```
CSTS physical address:
0x80000000 + 0x001C
```

# BAR Mapping Process

## Step 1 — Device advertises BAR

Device says:

> “I need 16 KB MMIO space.”

## Step 2 — BIOS/OS allocates address

Example:

```
BAR0 = 0x90000000
```

## Step 3 — CPU accesses device

Software performs MMIO (in C):

```
write32(0x90001000, tail_ptr);
```

This reaches the PCIe device.

# NVMe-Specific BAR Usage

An NVMe controller usually uses:

|BAR|Purpose|
|---|---|
|BAR0|Controller registers + doorbells|
|BAR1|Upper 32 bits if BAR0 is 64-bit|

Most NVMe devices expose a single 64-bit MMIO BAR.

# Important PCIe BAR Flags

A BAR contains metadata bits.

Example memory BAR layout:

|Bits|Meaning|
|---|---|
|Bit 0|0 = Memory BAR|
|Bits 2:1|32-bit or 64-bit|
|Bit 3|Prefetchable|
|Remaining bits|Base address|
# Prefetchable BAR

Indicates whether CPU may safely cache/prefetch reads.

Examples:

- GPU framebuffer → prefetchable
- Control registers → non-prefetchable

NVMe controller register BARs are usually non-prefetchable.

# Relationship to MMIO

BARs are the foundation of:

Memory-Mapped I/O

The CPU treats device registers like memory addresses.

Without BARs:

- CPU would not know where the device registers are mapped.

# Real Linux Example

Using (bash):

```
lspci -vv
```

You might see:

```
Region 0: Memory at 92000000 (64-bit, non-prefetchable) [size=16K]
```

Meaning:

- BAR0
- MMIO
- 64-bit
- 16 KB size

# Simplified Mental Model

You can think of a BAR as:

> “A window from CPU address space into the PCIe device.”

Example:

```
CPU Address Space
    |
    v
0x90000000 -----------------
| NVMe Controller Registers |
| Doorbells                 |
| Status Registers          |
-----------------------------
```

The BAR defines where that window starts.

# Key Takeaway

A PCIe BAR is:

- A configuration register in a PCIe device
- Used to request MMIO or I/O space
- Assigned an address by the OS/BIOS
- The mechanism enabling CPU ↔ device communication through mapped addresses

In NVMe, BARs are critical because they expose:

- controller registers,
- queue doorbells,
- status/control interfaces,
- interrupt configuration,  
    all through MMIO.

