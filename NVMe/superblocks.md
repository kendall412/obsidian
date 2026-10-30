
> A superblock in a NAND SSD is a logical grouping of multiple physical NAND erase blocks, typically distributed across multiple NAND dies, planes, or channels. The SSD controller uses superblocks to manage NAND storage more efficiently, particularly for parallel writing, garbage collection, wear leveling, and data placement. A superblock is not necessarily a physical structure inside a NAND chip. It is generally a controller-managed abstraction.

### 1. NAND storage hierarchy

To understand superblocks, first consider the organization of NAND flash.

![[nand flash org.png]]

A typical NAND hierarchy is:

|Level|Description|
|---|---|
|SSD|Complete storage device|
|Channel|Controller-to-NAND communication path|
|NAND package|Physical package containing one or more dies|
|NAND die|Independent NAND silicon device|
|Plane|Subdivision of a NAND die|
|Block|Physical NAND erase unit|
|Page|NAND programming and reading unit|
|Superblock|Logical grouping of blocks, often across dies or planes|

A superblock is a grouping across the hierarchy rather than necessarily being another physical level below a die.

### 2. Example of a superblock

Suppose an SSD contains four NAND dies, and each die contains 1,000 blocks. The controller could create superblocks by grouping one physical block from each die.

|Superblock|Die 0|Die 1|Die 2|Die 3|
|---|---|---|---|---|
|SB 0|Block 0|Block 0|Block 0|Block 0|
|SB 1|Block 1|Block 1|Block 1|Block 1|
|SB 2|Block 2|Block 2|Block 2|Block 2|
|SB 3|Block 3|Block 3|Block 3|Block 3|

Each superblock contains four physical NAND blocks.

#### Superblock capacity calculation

Assume:
- NAND page size = 16 KiB
- Pages per block = 256
- Blocks per superblock = 4

Physical block capacity:
```text
16 KiB * 256 = 4 MiB
```


Superblock capacity:
```text
4 MiB * 4 = 16 MiB
```


This is an illustrative raw capacity. Usable capacity may differ due to metadata, reserved space, parity, and other controller overhead.

### 3. Why do SSD controllers use superblocks?

#### A. Parallel NAND operations
One of the main benefits is that the controller can distribute operations across multiple NAND dies.

![[parallel writes across 4 dies.png]]

Instead of writing all data to a single NAND die, the controller can distribute writes across multiple dies, increasing the opportunity for parallel operations. Actual concurrency depends on NAND channels, die availability, and controller scheduling.

#### B. Garbage collection
Superblocks can serve as garbage collection units.

Suppose a superblock contains:
- 60% valid data
- 40% invalid data

To reclaim it, the controller may:
1. Identify the valid pages
2. Copy the valid data to another location
3. Update the Flash Translation Layer (FTL) mappings
4. Erase the physical blocks making up the superblock
5. Return the reclaimed blocks to the free-block pool

This is why superblock management affects write amplification.

#### C. Wear leveling

The controller can track erase counts and distribute writes among superblocks to prevent some physical blocks from wearing out much faster than others.

#### D. Bad block management

If one physical block becomes unusable, the controller may substitute another block or modify the superblock composition.

The exact mechanism is firmware-specific.

### 4. Superblock vs physical NAND block

|Characteristic|Physical block|Superblock|
|---|---|---|
|Structure|Physical NAND organization|Controller-defined logical grouping|
|Location|Within a NAND plane|May span multiple planes/dies|
|Erase operation|NAND erase granularity|May require multiple block erases|
|Size|Typically smaller|Usually larger|
|Managed by|NAND hardware and controller|SSD controller firmware|
|Main purpose|Store and erase NAND pages|Parallelism and allocation management|

Important: A superblock is not necessarily erased through one physical NAND command. The controller may issue multiple erase operations to erase its constituent blocks.

### 5. Superblock vs Virtual Superblock (VSB)
Some SSD firmware architectures introduce a Virtual Superblock (VSB).

|Term|Description|
|---|---|
|Physical NAND block|Actual erase block in NAND|
|Superblock|Group of physical blocks managed together|
|Virtual Superblock (VSB)|Firmware abstraction that represents a logical allocation unit and maps to physical blocks|
|Virtual Superblock Number (VSBN)|Identifier used by firmware to reference a VSB|

For example:
```
VSBN = 100
   |
   +-- Die 0 → Physical Block 120
   +-- Die 1 → Physical Block 145
   +-- Die 2 → Physical Block 130
   +-- Die 3 → Physical Block 160
```

The firmware can reference `VSBN 100` without requiring all the underlying physical block numbers to be identical. This provides flexibility for bad block replacement, wear leveling, and physical NAND management. These names and mapping conventions are vendor-specific, not standardized NVMe structures.

### 6. Relationship between superblocks and FDP Reclaim Units
Since you are studying NVMe Flexible Data Placement, the relationship between superblocks and Reclaim Units is particularly relevant. An FDP Reclaim Unit (RU) is a specification-defined unit of reclaimable nonvolatile storage. A superblock is an internal SSD firmware concept.

Depending on the firmware architecture, an RU could correspond to:
- One superblock.
- Multiple superblocks.
- Another controller-defined physical allocation unit.

For example:
![[superblock & RU.png]]


For example, if a superblock provides 16 MiB of usable storage and an RU consists of four such superblocks:

```text
RU Capacity = 4 * 16 MiB = 64 MiB
```

The FDP Configurations log reports the nominal RU size through RUNS, but it does not reveal the internal physical superblock mapping.

### 7. Superblocks in SSD firmware validation

When debugging NAND allocation, garbage collection, or FDP placement, useful firmware diagnostics include:

|Diagnostic|What it tells you|
|---|---|
|Superblock ID / VSBN|Which logical allocation unit is used|
|Physical block mapping|Which NAND blocks form the superblock|
|Superblock state|Free, open, closed, reclaiming, bad|
|Valid page count|Amount of live data|
|Invalid page count|Amount of obsolete data|
|Erase count|NAND wear|
|Reclaim Unit association|Which FDP RU uses the superblock|
|RUH association|Which placement path directs writes to it|
|GC victim selection|Which superblock is selected for reclamation|

For example, if FDP writes with PID `0x0001` unexpectedly cause high write amplification, firmware traces could reveal whether the associated RU is sharing superblocks with data having different lifetimes, or whether garbage collection is relocating substantial amounts of valid data.

Key takeaway: A NAND superblock is a controller-managed grouping of physical erase blocks, often across multiple NAND dies. It enables parallelism and simplifies allocation, garbage collection, and wear leveling. In an FDP-capable SSD, superblocks may be used internally to implement Reclaim Units, but the NVMe specification does not require a fixed one-to-one relationship.

