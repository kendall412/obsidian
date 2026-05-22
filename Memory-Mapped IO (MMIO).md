
> Memory-Mapped I/O (MMIO) is a technique where **hardware device registers are mapped into the CPU’s memory address space**. This allows software to communicate with hardware devices using ordinary memory read/write instructions. Instead of using special I/O instructions, **the CPU accesses device registers as if they were normal memory addresses.

# Core Idea

With MMIO:

```
Certain physical memory addresses
do not point to RAM.
They point to hardware registers.
```

Example:

|Address|Actually Refers To|
|---|---|
|0x80000000|NVMe controller register|
|0xFEC00000|Interrupt controller|
|0xA0000000|GPU framebuffer|

So when software writes to those addresses, it is actually controlling hardware.

# Simple Example

Suppose an NVMe SSD has:

```
BAR0 = 0x90000000
```

And the controller’s status register is at:

```
Offset = 0x1C
```

Then:

```
MMIO Address = 0x90000000 + 0x1C             = 0x9000001C
```

Software reads it like memory (in C):

```
status = *(volatile uint32_t *)0x9000001C;
```

But the CPU is actually reading from the NVMe controller hardware.

# Why MMIO Exists

The CPU needs to communicate with devices:

- SSDs
- GPUs
- NICs
- USB controllers
- FPGA cards

MMIO provides a standardized method.

# MMIO vs Normal RAM

## Normal Memory Access

```
CPU → DRAM controller → RAM chips
```

## MMIO Access

```
CPU → PCIe root complex → PCIe device registers
```

The CPU issues a memory transaction in both cases, but the destination differs.

# Relationship Between BAR and MMIO

A PCIe device exposes resources through:

Base Address Register

The BAR tells the OS:

> “Map my device registers into memory space here.”

Then MMIO is used to access those mapped addresses.

# Example Flow (NVMe)

## Step 1 — Device exposes BAR

NVMe device says:

```
Need 16 KB MMIO space
```

## Step 2 — OS assigns address

```
BAR0 = 0x90000000
```

## Step 3 — Driver accesses MMIO registers

```
write32(0x90001000, sq_tail);
```

This write becomes a PCIe Memory Write TLP.

# MMIO Accesses Become PCIe Transactions

When CPU executes (in C):

```
*(volatile uint32_t *)addr = value;
```

hardware converts it into:

```
PCIe Memory Write Request
```

Similarly:

```
x = *(volatile uint32_t *)addr;
```

becomes:

```
PCIe Memory Read Request
```

# Typical MMIO Registers

Devices expose:

|Register Type|Purpose|
|---|---|
|Control|Start/stop hardware|
|Status|Device state|
|Doorbells|Notify hardware|
|Interrupt masks|Interrupt control|
|DMA configuration|Memory transfer setup|

# MMIO in NVMe

Important NVMe registers accessed through MMIO:

| Register  | Purpose                     |
| --------- | --------------------------- |
| CAP       | Controller capabilities     |
| CC        | Controller configuration    |
| CSTS      | Controller status           |
| AQA       | Admin queue attributes      |
| ASQ       | Admin submission queue base |
| ACQ       | Admin completion queue base |
| Doorbells | Queue notifications         |

# Why `volatile` Is Used

MMIO registers can change independently of software.

Compiler optimizations are dangerous.

Example (in C):

```
volatile uint32_t *reg;
```

Without `volatile`, compiler may:

- cache values,
- remove reads,
- reorder accesses.

That would break hardware communication.

# MMIO Is Usually Uncacheable

Device registers are generally mapped as:

```
Uncacheable (UC)
```

because caching hardware registers would produce stale or invalid behavior.


# MMIO vs Port-Mapped I/O (PMIO)

## MMIO

Uses memory instructions:

```
load/store
```

Modern systems prefer this.

## PMIO

Uses special CPU instructions (in asm):

```
IN
OUT
```

Legacy x86 mechanism.

PCIe/NVMe primarily use MMIO.

# MMIO Read/Write Latency

MMIO is much slower than RAM because accesses traverse:

```
CPU
→ interconnect
→ PCIe root complex
→ PCIe link
→ device
```

So drivers avoid excessive MMIO reads.

# Memory Map Example

```
System Physical Address Space

0x00000000 - 0x7FFFFFFF   RAM
0x80000000 - 0x80003FFF   NVMe MMIO BAR
0x90000000 - 0x9FFFFFFF   GPU framebuffer
0xFEC00000 - 0xFEC00FFF   APIC
```

Not all “memory addresses” refer to RAM.

# MMIO Write Example: NVMe Doorbell

Driver submits command into Submission Queue in RAM.

Then rings doorbell (in C):

```
*(volatile uint32_t *)(bar0 + doorbell_offset) = new_tail;
```

This tells the NVMe controller:

> “New command available.”

# Important MMIO Characteristics

| Property             | Description                  |
| -------------------- | ---------------------------- |
| Address-based        | Accessed like memory         |
| Hardware-backed      | Not normal RAM               |
| Usually uncached     | Prevent stale data           |
| Ordered carefully    | Reordering can break devices |
| Used by PCIe devices | Standard mechanism           |

# Simplified Mental Model

MMIO is essentially:

```
CPU memory accesses redirected to hardware devices
```

instead of DRAM.

# Key Takeaway

MMIO lets the CPU control hardware devices using normal memory reads/writes.

In PCIe systems:

- BARs define where device registers are mapped,
- MMIO accesses those registers,
- CPU memory operations become PCIe transactions to the device.

This is the foundation of how:

- NVMe SSDs,
- GPUs,
- NICs,
- FPGA accelerators,  
    communicate with software.