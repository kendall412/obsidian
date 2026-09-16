#vsb #vsbn
A **Virtual Super Block (VSB)** in NAND/SSD terminology is generally a **logical grouping of multiple physical NAND blocks—often spread across dies, planes, or channels—that the SSD controller treats as one larger allocation/erase-management unit**.

The exact meaning is **vendor/controller-specific**; “Virtual Super Block” is not a universal raw-NAND structure defined identically across all SSDs. But the concept is important in SSD firmware and FTL design.

### Start with the NAND hierarchy

A simplified NAND organization looks like:

```text
SSD
 │
 ├── NAND Package
 │     │
 │     ├── Die 0
 │     │    ├── Plane 0
 │     │    │    ├── Block 0
 │     │    │    ├── Block 1
 │     │    │    └── ...
 │     │    └── Plane 1
 │     │
 │     └── Die 1
 │          └── ...
 │
 └── Other NAND packages...
```

A physical NAND **block** contains many pages:

```text
Physical Block
┌──────────────────┐
│ Page 0           │
│ Page 1           │
│ Page 2           │
│ ...              │
│ Page N           │
└──────────────────┘
```

==Pages are programmed/read, while erase normally happens at the physical block level.== A VSB exists **above these physical blocks as a controller abstraction**.

### Why create a Virtual Super Block?

An SSD has many NAND resources operating in parallel:

```text
Channel 0     Channel 1     Channel 2     Channel 3
   │             │             │             │
  Die 0         Die 0         Die 0         Die 0
   │             │             │             │
 Block 27      Block 41      Block 16      Block 32
   └─────────────┬─────────────┬─────────────┘
                 │
                 ▼
         Virtual Super Block
```

Instead of firmware managing those blocks completely independently, it can associate them with one larger logical structure.

For example:

```text
VSB #10

Physical member             Location
------------------------------------------------
Block 27                    Channel 0 / Die 0
Block 41                    Channel 1 / Die 0
Block 16                    Channel 2 / Die 0
Block 32                    Channel 3 / Die 0
```

The [[FTL (Flash Translation Layer)]] can then allocate/write data across the members to exploit NAND parallelism.

### Why "virtual"?

Because there isn't necessarily a physical NAND structure called a VSB.
The NAND itself understands things such as:

```text
Die
  ↓
Plane
  ↓
Block
  ↓
Page
  ↓
Cell
```

The controller firmware creates the additional abstraction:

```text
                 FTL
                  │
                  ▼
        Virtual Super Block
          /      |      \
         /       |       \
        ▼        ▼        ▼
Physical     Physical    Physical
Block        Block       Block
   │            │           │
 NAND         NAND        NAND
```

So **virtual** means that the grouping exists primarily in the controller's metadata/FTL rather than being a single physical NAND block.

### VSB & parallelism

This is one of the major reasons such structures are useful. Suppose one NAND operation has limited throughput. If the controller stripes data across several dies,

```text
Incoming writes
      │
      ▼
┌───────────────────────┐
│ Virtual Super Block   │
└───────────────────────┘
      │
 ┌────┼────┬────┐
 ▼    ▼    ▼    ▼
Die0 Die1 Die2 Die3
 │    │    │    │
PGM  PGM  PGM  PGM
```

multiple NAND program operations can be in flight, subject to the NAND/controller architecture. This is somewhat analogous to **striping**, although you shouldn't assume a VSB is literally RAID; the exact implementation depends on the SSD.

### VSB vs physical NAND block

The distinction is:

|Physical NAND block|Virtual Super Block|
|---|---|
|Physically exists in NAND|Controller/FTL abstraction|
|Belongs to a specific die/plane|May encompass blocks across multiple resources|
|NAND erase unit|Often used as a larger FTL management/allocation unit|
|Contains physical pages|Represents/group maps physical blocks/pages|
|NAND hardware concept|SSD firmware concept|

For example, imagine:

```text
Die 0                 Die 1
┌──────────┐          ┌──────────┐
│ Block 20 │          │ Block 35 │
└──────────┘          └──────────┘
      \                   /
       \                 /
        └───────┬───────┘
                ▼
             VSB 7
```

Firmware might have metadata conceptually similar to:

```c
struct virtual_super_block {
    block_t die0_block;
    block_t die1_block;
    ...
};
```

Real firmware structures are generally considerably more complicated.

### Connection to the FTL

For SSD validation, the important relationship is:

```text
Host
 │
 │ LBA writes
 ▼
NVMe Controller
 │
 ▼
FTL
 │
 ├── LBA → physical mapping
 │
 ├── allocation
 │
 ├── garbage collection
 │
 ├── wear leveling
 │
 └── VSB management
 │
 ▼
Virtual Super Block
 │
 ├── NAND block
 ├── NAND block
 ├── NAND block
 └── NAND block
```

VSBs can therefore be involved in **write allocation, garbage collection, wear leveling, bad-block handling, and NAND parallelism**.

One important correction to the wording in your question: a VSB usually isn't best thought of as something **“inside a NAND die.”** A physical block is inside a NAND die. A **Virtual Super Block is normally a controller-level abstraction that groups physical NAND resources**, and depending on the particular architecture those resources may span planes/dies/channels.

If you're seeing **“VSB” in a specific Solidigm/SK hynix/controller debug log or NAND address structure**, its definition may be more specific than the generic SSD meaning above. In that case, show me the field/structure where `VSB` appears and I can map it to **channel → CE/LUN → die → plane → block → page**.

## Why does VSB exists?

The **Virtual Super Block (VSB) abstraction exists mainly because an SSD controller does not want to manage every physical NAND block independently**. It creates a larger logical allocation unit that lets the FTL coordinate many NAND resources efficiently.

The central idea is:

```text
Host writes
    │
    ▼
   FTL
    │
    ▼
Virtual Super Block
    │
    ├── Physical NAND block
    ├── Physical NAND block
    ├── Physical NAND block
    └── Physical NAND block
```

Those physical blocks may reside on different NAND resources, depending on the controller architecture.

### 1. Exploit NAND parallelism

An SSD contains many NAND dies/planes/channels. If the controller wrote to only one physical block at a time, much of that hardware would sit idle.

Instead, a VSB can provide a logical unit over multiple resources:

```text
                 VSB 12
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
     Die 0       Die 1       Die 2
       │           │           │
    Block 50    Block 71    Block 38
       │           │           │
     PROGRAM     PROGRAM     PROGRAM
```

The controller can schedule operations across these resources concurrently where the hardware permits it.
This is one of the ways SSDs obtain much higher throughput than a single NAND die can provide.

---

### 2. Give the FTL a convenient allocation unit

Imagine a large SSD containing thousands of NAND blocks across many dies. Without an abstraction, firmware would constantly deal directly with:

```text
Channel
  ↓
CE
  ↓
LUN/Die
  ↓
Plane
  ↓
Physical Block
  ↓
Page
```

Instead, the FTL can work at a higher level:

```text
VSB 0
VSB 1
VSB 2
...
VSB 1000
```

Then lower-level NAND management translates a VSB and offsets within it into the appropriate physical resources.

Conceptually:

```text
FTL
 │
 │ allocate VSB 25
 ▼
VSB management
 │
 ├── Die 0 → Block 103
 ├── Die 1 → Block 105
 ├── Die 2 → Block 101
 └── Die 3 → Block 107
```

This separates **logical media management** from the exact physical NAND geometry.

---

### 3. Garbage collection becomes easier to manage

Because NAND cannot overwrite an already programmed page directly, SSDs eventually perform garbage collection.

Suppose a VSB contains:

```text
VSB 20

[Valid]
[Invalid]
[Invalid]
[Valid]
[Invalid]
[Valid]
...
```

The controller can:

```text
        Old VSB
           │
           │ copy valid data
           ▼
        New VSB
           │
           ▼
   Update FTL mapping
           │
           ▼
Erase/reclaim old physical blocks
```

Grouping storage into larger management units gives firmware a practical structure for tracking things such as valid data, free space, erase/reclaim state and GC candidates.

---

### 4. Wear leveling

Physical NAND blocks have finite program/erase endurance.

Suppose firmware tracks something conceptually like:

```text
VSBN       Erase count
----------------------
100          1,502
101          1,487
102          8,921   ← much more worn
103          1,510
```

The FTL/media-management firmware can use its VSB-level metadata when deciding where to allocate new data and which physical resources to rotate.

The exact wear-leveling algorithm is controller-specific, but the abstraction makes large-scale media management easier.

---

### 5. Bad-block management

This is another important reason not to expose physical geometry directly to higher FTL layers.
Suppose a logical grouping originally involves:

```text
VSB 55

Die 0 → Block 100
Die 1 → Block 100
Die 2 → Block 100   ← bad
Die 3 → Block 100
```

The controller's media-management layer may be able to accommodate that defect according to its particular architecture—for example through replacement/remapping mechanisms—while higher software layers continue referring to the logical VSB.

Conceptually:

```text
                 VSBN 55
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
      Die 0       Die 1       Die 2
      Blk100      Blk100       BAD
                                │
                                ▼
                           replacement/
                           remapping
```

The exact mechanism varies considerably between SSD implementations.

---

### 6. Hide NAND geometry from higher firmware layers

This becomes especially useful when SSD generations change.
One NAND generation might have:

```text
2 planes/die
X pages/block
Y dies/package
```

while another has:

```text
4 planes/die
different pages/block
different dies/package
```

You don't necessarily want every FTL algorithm rewritten around those details.

Instead:

```text
                Higher FTL
                    │
                    │
                   VSB
                    │
           Media management
                    │
         ┌──────────┴──────────┐
         ▼                     ▼
     NAND Gen A            NAND Gen B
```

The abstraction boundary can isolate some physical-media differences from higher layers.

---

### The important distinction

Don't think of a VSB as simply a **bigger NAND block physically manufactured inside the die**.

Think of it as:

> **A firmware-defined logical storage/management unit representing a collection of physical NAND resources.**

So you effectively have two levels:

```text
Logical/controller level
────────────────────────

        VSBN = 25
            │
            ▼
      Virtual Super Block
            │
            │ mapping/organization
            ▼

Physical NAND level
────────────────────────

Channel → Die → Plane → Block → Page
```

This also explains why **VSBN (Virtual Super Block Number)** is useful. Instead of higher-level firmware passing around a long physical NAND address every time, it can identify a managed unit by something like:

```text
VSBN = 0x125
```

and the lower-level media-management code knows what physical NAND resources that VSB represents.

For SSD validation work, a useful next concept is **how `VSBN + page/wordline offset` can turn into a physical NAND address (channel/CE/LUN/plane/block/page)**. That is where the VSB abstraction starts connecting directly to NAND commands and debug traces.