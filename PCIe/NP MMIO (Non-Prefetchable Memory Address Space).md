
> In PCIe, Non-Prefetchable Memory-Mapped I/O (NP MMIO) is a region of the system's memory address space assigned to a PCIe device where the CPU accesses device registers or memory using normal memory read and write operations, but *the system must not speculatively read or prefetch data from that region.* NP MMIO is commonly used for device control registers, status registers, and other hardware resources that have side effects when accessed.

### 1. What is MMIO?
MMIO stands for Memory-Mapped I/O. It allows the CPU to communicate with a PCIe device by assigning the device a range of physical memory addresses.

For example, suppose an NVMe SSD has registers mapped to:

`0x80000000 – 0x80003FFF`

The CPU can access those registers using memory operations.

![[mmio.png]]

When the CPU reads or writes an address within this range, the host bridge/root complex routes the operation to the appropriate PCIe device. The PCIe device interprets the address as an offset within one of its Base Address Register (BAR) regions.

### 2. What does Non-Prefetchable mean?
*Prefetching is an optimization in which a processor or memory subsystem reads data before software explicitly requests it.* This can improve performance for ordinary memory, but it can cause problems with hardware registers.

Consider a device status register at address `0x80000010`.

Suppose reading this register clears an interrupt status bit.
1. The device sets an interrupt status bit
2. The CPU is expected to read the register to acknowledge the event
3. If the system speculatively reads the register before software requests it, the status bit might be cleared unexpectedly
4. The software may then miss the event

Non-Prefetchable MMIO prevents speculative or additional reads that could produce unintended hardware behavior.

The PCIe prefetchability attribute also concerns whether reads can have side effects and whether accesses can be safely combined or otherwise optimized. It is not simply a CPU cache enable/disable bit.

### 3. Prefetchable vs Non-Prefetchable MMIO

|Characteristic|Prefetchable MMIO|Non-Prefetchable MMIO|
|---|---|---|
|Speculative reads|Permitted|Not permitted|
|Read side effects|Must not have side effects|May have side effects|
|Typical usage|Device memory, frame buffers|Control/status registers|
|Read merging|May be permitted|Must preserve required access semantics|
|CPU caching|Depends on memory attributes|Typically mapped uncached|
|Example|GPU frame buffer|NVMe controller registers|

An important distinction is that prefetchable does not automatically mean cacheable. CPU memory types, such as UC and WC on x86, separately control caching, ordering, and write-combining behavior.

### 4. How does PCIe identify NP MMIO?

PCIe devices expose Base Address Registers (BARs) in their PCI configuration space.
For a Memory BAR, the lowest four bits describe the BAR's properties

![[BAR setup.png]]

![[memory bar lower bits.png]]

- Bit 0 = 0: Memory Space BAR
- Bits 2:1 = 10b: 64-bit Memory BAR
- Bit 3 = 0: Non-Prefetchable
- Bit 3 = 1: Prefetchable

For example, a BAR reporting:

`0x80000004`

has these properties:
- Bit 0 = 0 → Memory Space
- Bits 2:1 = 10b → 64-bit BAR
- Bit 3 = 0 → Non-Prefetchable
    
The base address represented by the low BAR DWORD is `0x80000000`; the upper BAR DWORD supplies the remaining address bits.

### 5. NP MMIO in an NVMe SSD
NVMe controllers typically expose their controller register space through a non-prefetchable PCIe Memory BAR.

Examples include:

|Register|Offset|Purpose|
|---|---|---|
|CAP|0x0000|Controller Capabilities|
|VS|0x0008|Version|
|CC|0x0014|Controller Configuration|
|CSTS|0x001C|Controller Status|
|AQA|0x0024|Admin Queue Attributes|
|ASQ|0x0028|Admin Submission Queue Base Address|
|ACQ|0x0030|Admin Completion Queue Base Address|
|Doorbells|0x1000+|SQ Tail and CQ Head updates|

For example, when the host writes to the NVMe Submission Queue Tail Doorbell register:
1. The CPU performs an MMIO write to the doorbell address
2. The Root Complex translates that operation into a PCIe Memory Write Request TLP
3. The NVMe controller receives the write and updates its internal doorbell state
4. The controller can then process newly submitted commands

The doorbell region is non-prefetchable because it represents hardware control state, not ordinary RAM.

Note that the Submission Queues and Completion Queues themselves are usually located in host system memory. They are distinct from the controller's BAR-mapped registers.

### 6. Does NP MMIO affect PCIe TLPs?
Yes, but it is important to distinguish PCIe address-space attributes from PCIe TLP attributes.

A CPU access to NP MMIO typically becomes one of these transactions:
- Memory Read Request TLP
- Memory Write Request TLP

There is no separate PCIe TLP type called an "NP MMIO TLP."

Also, *the term Non-Prefetchable should not be confused with PCIe's Non-Posted transactions.*

|Term|Meaning|
|---|---|
|Non-Prefetchable MMIO|A memory region that cannot be speculatively read|
|Non-Posted Request|A PCIe request requiring a Completion, such as a Memory Read|
|Posted Request|A PCIe request that does not require a Completion, such as a Memory Write|

A write to a Non-Prefetchable MMIO region is still normally a Posted Memory Write transaction.

### 7. How to identify NP MMIO in Linux

Use:
```
sudo lspci -vv -s 01:00.0
```

Example output:
```
Region 0: Memory at 80000000 (64-bit, non-prefetchable) [size=16K]
```

This indicates:
- The device has a memory BAR
- Its assigned base address is `0x80000000`
- It supports a 64-bit address
- It is non-prefetchable
- The allocated address space is 16 KB

The operating system can map this BAR so that its driver can access the device's hardware registers.

Key takeaway: NP MMIO is a PCIe device address region used primarily for hardware registers where reads must occur only when explicitly requested, because reading may have side effects. It is particularly important for NVMe controller registers, doorbells, and other device control interfaces.