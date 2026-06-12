# Three Different Memory Areas

During NVMe operation, the host allocates several different types of memory:

```
1. Submission Queues (SQ)
2. Completion Queues (CQ)
3. Data Buffers
```

These have different lifetimes.

# During Initialization

The NVMe driver creates:

```
Admin SQ
Admin CQ
I/O SQs
I/O CQs
```

Example:

```
Host Memory

+-------------+
| Admin SQ    |
+-------------+

+-------------+
| Admin CQ    |
+-------------+

+-------------+
| IO SQ 1     |
+-------------+

+-------------+
| IO CQ 1     |
+-------------+
```

These queues typically remain allocated for the life of the controller.

# Data Buffers Are Different

Consider a Read command:

```
Read LBA 1000
Length = 16 KB
```

The host needs a place to store the data.

It allocates or uses an existing buffer:

```
+----------------+
| Data Buffer    |
| 16 KB          |
+----------------+
```

Then builds the NVMe command:

```
Opcode = Read
PRP1   = buffer address
```

and submits it.

# Write Example

Suppose an application wants to write:

```
"Hello World"
```

The data already exists in memory:

```
Application Buffer

+----------------+
| Hello World    |
+----------------+
```

The NVMe driver uses that buffer (or maps it for DMA).

Then:

```
Write Command
    |
PRP1 --> Buffer Address
```

and submits the command.

The buffer exists **before** the command is placed into the SQ.

# Read Example

For a Read:

```
Application requests 4 KB read
```

The OS allocates:

```
+----------------+
| Empty Buffer   |
+----------------+
```

Then submits:

```
Read Command
```

with:

```
PRP1 = buffer address
```

The controller DMA-writes the data into that buffer.

# Timeline

## Initialization

```
Driver Load
     |
Create SQs
Create CQs
     |
Controller Ready
```

No user-data buffers are created here.

## Later: Read Command

```
Application requests read
          |
Allocate buffer
          |
Build command
          |
Insert SQ entry
          |
Ring doorbell
```

# Why Not Allocate All Buffers During Initialization?

Imagine a 4 TB SSD.

Allocating buffers for every possible I/O would require enormous memory.

Instead:

```
Allocate buffer
Use buffer
Free/reuse buffer
```

as needed.

# PRP Example

Suppose the host allocates:

```
Virtual Address:
0x7fff100000
```

which maps to:

```
Physical Address:
0x12345000
```

The command contains:

```
PRP1 = 0x12345000
```

The controller DMA accesses:

```
0x12345000
```

directly.

# What About High-Performance Systems?

Some systems pre-allocate large pools of DMA buffers.

Example:

```
NVMe Driver Startup

Allocate 1024 DMA Buffers
```

Then later:

```
Read Request
     |
Borrow buffer from pool
     |
Submit command
     |
Return buffer to pool
```

Even here, the buffers are not tied to queue creation.

The queues already exist.

# Relationship Between SQ Entry and Data Buffer

```
Host Memory

+----------------+
| Data Buffer    |
+----------------+
         ^
         |
      PRP1

+----------------+
| SQ Entry       |
| Opcode=Read    |
| PRP1=addr      |
+----------------+
```

The SQ entry contains pointers to the data buffer.

The buffer itself is separate from the queue.

# Summary

|Object|Created When?|Lifetime|
|---|---|---|
|Admin SQ/CQ|Controller initialization|Persistent|
|I/O SQ/CQ|Controller initialization|Persistent|
|Read buffer|Before Read command submission|Per I/O or from buffer pool|
|Write buffer|Before Write command submission|Per I/O or from buffer pool|
|PRP/SGL entries|When building the command|Per command|

> So, for normal NVMe I/O, the **host creates or obtains the data buffer before submitting the command to the Submission Queue**. The **Submission Queues and Completion Queues themselves are typically created once during initialization and reused for many commands.**

---
# Different Types of buffers in NVMe

There are several different kinds of memory buffers involved in an NVMe system. Some belong to the **host**, some belong to the **controller**, and some are shared.

# High-Level View

```
Host Memory (DRAM)
│
├── Submission Queues
├── Completion Queues
├── Data Buffers
├── PRP Lists
├── SGL Descriptors
├── Metadata Buffers
└── Driver Buffers

        PCIe

NVMe Controller
│
├── Controller DRAM
├── Read Cache
├── Write Cache
├── FTL Mapping Cache
├── Admin Buffers
└── Controller Memory Buffer (optional)
```

# 1. Submission Queue (SQ) Buffer

Allocated by the host.

Contains NVMe commands.

```
Submission Queue

+------------+
| Read Cmd   |
+------------+
| Write Cmd  |
+------------+
| Flush Cmd  |
+------------+
```

Purpose:

```
Host → Controller command communication
```

Created during initialization.

# 2. Completion Queue (CQ) Buffer

Allocated by the host.

Contains completion entries.

```
Completion Queue

+------------+
| CQE        |
+------------+
| CQE        |
+------------+
```

Purpose:

```
Controller → Host completion communication
```

Created during initialization.

# 3. Read Data Buffer

Allocated by the host before a Read command.

Initially empty:

```
+----------------+
| Empty Buffer   |
+----------------+
```

Controller DMA-writes data into it.

```
SSD ---> Host Buffer
```


# 4. Write Data Buffer

Contains data to be written.

Example:

```
+----------------+
| User Data      |
+----------------+
```

Controller DMA-reads from it.

```
Host Buffer ---> SSD
```

# 5. PRP Buffer / PRP List

PRP = Physical Region Page.

Used when data spans multiple pages.

Example:

```
PRP1 --> Page 0

PRP List

+----------------+
| Page 1 Addr    |
+----------------+
| Page 2 Addr    |
+----------------+
| Page 3 Addr    |
+----------------+
```

Purpose:

```
Describe locations of data buffers
```


# 6. SGL Buffer

SGL = Scatter Gather List.

Alternative to PRPs.

Used when data is fragmented.

```
SGL

+----------------+
| Addr A         |
| Length A       |
+----------------+
| Addr B         |
| Length B       |
+----------------+
```

Purpose:

```
Describe scattered memory regions
```

# 7. Metadata Buffer

Used when namespace metadata exists.

Example:

```
Data Buffer
+----------------+
| 4096 bytes     |
+----------------+

Metadata Buffer
+----------------+
| CRC            |
| Ref Tag        |
+----------------+
```

Referenced by:

```
MPTR
```

(Metadata Pointer)

# 8. Admin Command Buffers

Used during controller management.

Examples:

```
Identify
Get Log Page
Firmware Download
```

For an Identify command:

```
Controller
     |
DMA
     v

+----------------+
| Identify Data  |
| 4096 bytes     |
+----------------+
```

# 9. Controller DRAM

Located inside the SSD.

Used for:

```
FTL mappings
Queue state
Wear leveling data
Garbage collection
```

Example:

```
LBA 1000 → NAND Page 12345
```

Usually invisible to the host.

# 10. Read Cache

Controller-side buffer.

```
NAND
  |
  v

Read Cache
```

Stores recently read data.

Purpose:

```
Reduce NAND reads
```

# 11. Write Cache

Controller-side buffer.

```
Host Data
    |
    v

Write Cache
```

Stores incoming writes before NAND programming.

Purpose:

```
Improve write performance
```

May be:

- DRAM
- SRAM
- HMB-backed

# 12. Controller Memory Buffer (CMB)

Optional NVMe feature.

Controller exposes onboard memory to host.

```
Host
  |
PCIe
  |
CMB
```

Can store:

```
Submission Queues
Completion Queues
Data Buffers
```

instead of host DRAM.

Registers:

```
CMBLOC
CMBSZ
```

# 13. Host Memory Buffer (HMB)

Used by DRAM-less SSDs.

Host lends memory to SSD.

```
Host DRAM
     |
     v

HMB
```

Controller uses it for:

```
FTL cache
Read cache
Metadata cache
```

Configured through:

```
Set Features
Feature ID = HMB
```

# 14. Firmware Download Buffer

Used during firmware updates.

```
Firmware Image
     |
DMA
     v

Firmware Download Buffer
```

Commands:

```
Firmware Download
Firmware Commit
```

# Complete Data Path Example

### Write

```
Application Buffer
        |
        v
Write Buffer
        |
PRP/SGL
        |
SQ Entry
        |
PCIe DMA
        |
Write Cache
        |
FTL
        |
NAND
```

### Read

```
NAND
   |
FTL
   |
Read Cache
   |
PCIe DMA
   |
Read Buffer
   |
Application
```

# Most Important Buffers for NVMe Driver Engineers

```
Host Side

1. Submission Queue Buffer
2. Completion Queue Buffer
3. Read Data Buffer
4. Write Data Buffer
5. PRP Lists
6. SGL Lists
7. Metadata Buffer
```

# Most Important Buffers for NVMe Firmware Engineers

```
Controller Side

1. FTL Mapping Cache
2. Write Cache
3. Read Cache
4. HMB Cache
5. CMB Memory
6. NAND Metadata Buffers
```

These buffers work together to move data efficiently between applications, host memory, the NVMe controller, and NAND flash.