
> In NVMe, **PRPs (Physical Region Pages)** and **SGLs (Scatter-Gather Lists)** are two different ways for the host to tell the controller where the data buffer resides in host memory.

The NVMe command contains either:

- PRP pointers, or
- SGL descriptors

but not both for the same data transfer.

# High-Level Comparison

|Feature|PRP|SGL|
|---|---|---|
|Defined in original NVMe spec|Yes|Added later|
|Most common on PCs|Yes|Less common|
|Best for|Page-oriented memory|Complex scatter-gather buffers|
|Used by Linux/Windows NVMe drivers|Usually|Sometimes|
|Command fields used|PRP1, PRP2|SGL descriptor|
|Controller support required|Mandatory|Optional|

# [[PRP (Physical Region Page)]]

PRP is the original NVMe mechanism.

The command contains:

```
PRP1
PRP2
```

Example Read command:

```
+----------------+
| PRP1           |
+----------------+
| PRP2           |
+----------------+
```

These point to host memory pages.

## Small Transfer Example

Read 4 KB:

```
Host Buffer

0x10000000
+-----------+
| 4 KB Page |
+-----------+
```

Command:

```
PRP1 = 0x10000000
PRP2 = 0
```

Only one page is needed.

## Larger Transfer Example

Read 16 KB:

```
Page0
Page1
Page2
Page3
```

Command:

```
PRP1 -> Page0
PRP2 -> PRP List
```

The PRP List contains:

```
Page1
Page2
Page3
```

## Why PRP Exists

PRP is optimized for:

- Page-based memory
- Operating systems
- DMA engines

Most NVMe SSD traffic uses PRPs.

# [[SGL (Scatter-Gather List)]]

SGL is more flexible.
Instead of page pointers, it uses descriptors.

Example:

```
Descriptor
{
    Address
    Length
    Type
}
```

## SGL Example

Suppose the buffer is fragmented:

```
0x10000000  4 KB
0x20000000  8 KB
0x30000000  2 KB
```

SGL can directly describe:

```
Descriptor 1
Address = 0x10000000
Length  = 4096

Descriptor 2
Address = 0x20000000
Length  = 8192

Descriptor 3
Address = 0x30000000
Length  = 2048
```

No PRP list construction required.

# Where Does the Command Indicate PRP or SGL?

The command contains:

```
PSDT
(PRP or SGL for Data Transfer)
```

field in CDW0.

The controller checks [[PRP or SGL for Data Transger (PSDT)]].

### PSDT = 0

```
Use PRP
```

### PSDT = 1

```
Use SGL
```

# When Is PRP Used?

PRP is typically used when:

### 1. Standard PC Operating Systems

Most desktop systems:

- Windows
- Linux
- FreeBSD

primarily use PRPs.

Example:

```
NVMe SSD in laptop
```

Almost always PRP-based.

### 2. Simple Contiguous Buffers

```
Buffer
+---------+
| 128 KB  |
+---------+
```

PRPs efficiently describe the memory.

### 3. Consumer SSDs

Most consumer NVMe workloads:

```
Filesystem reads
Filesystem writes
```

use PRPs.

# When Is SGL Used?

SGL becomes useful when memory is highly fragmented.

### 1. Enterprise Storage

Examples:

- RAID controllers
- Storage appliances
- High-performance networking

Memory may be scattered:

```
Buffer A
Buffer B
Buffer C
Buffer D
```

SGL directly describes them.

### 2. NVMe over Fabrics (NVMe-oF)

SGL is heavily used in:

- RDMA
- Fibre Channel
- TCP transports

because buffers may already exist as scatter-gather lists.

### 3. Embedded Systems

Some embedded DMA architectures naturally produce:

```
Scatter-Gather Buffers
```

SGL avoids converting them into PRP lists.

# Example: 64 KB Read Using PRP

```
Read 64 KB
```

Pages:

```
Page0
Page1
Page2
...
Page15
```

Command:

```
PRP1 -> Page0

PRP2 -> PRP List
```

PRP List:

```
Page1
Page2
Page3
...
Page15
```

Controller walks the PRP list.

# Example: 64 KB Read Using SGL

Command contains:

```
SGL Segment Descriptor
```

which points to:

```
Descriptor1
Descriptor2
Descriptor3
...
```

Each descriptor specifies:

```
Address
Length
Type
```

The controller walks the SGL descriptors.

# Controller Capability

Not every controller supports SGL.

The Identify Controller data structure contains:

```
SGLS
(SGL Support field)
```

which tells the host what SGL types are supported.

The host checks this before using SGL.

# Real-World NVMe SSD Firmware

For a typical consumer SSD:

```
Windows
   |
NVMe Driver
   |
PRP1 / PRP2
   |
SSD Controller
```

PRP handling is exercised constantly.

SGL support may exist but be used infrequently.

For an enterprise NVMe SSD:

```
NVMe-oF
Storage Appliance
RDMA
```

SGL usage is much more common.

# Summary

**PRP** and **SGL** solve the same problem: describing host memory buffers for DMA transfers.

- **PRP** uses page pointers (`PRP1`, `PRP2`, and PRP Lists).
- **SGL** uses flexible descriptors (`Address`, `Length`, `Type`).

Typical usage:

```
Consumer NVMe SSDs:
    Mostly PRP

Enterprise Storage:
    PRP + SGL

NVMe over Fabrics:
    Often SGL
```

A useful rule of thumb is:

> If the host buffer is simple and page-oriented, NVMe usually uses **PRP**. If the buffer is highly fragmented or already represented as scatter-gather segments, NVMe often uses **SGL**.


