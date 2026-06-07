
> The **CAP (Controller Capabilities)** register is one of the most important registers in an NVMe controller. The Controller Capabilities (CAP) register is a key memory-mapped register in an NVMe controller that provides host software with **read-only information about the fundamental features and capabilities of the device**. It is a **64-bit read-only register** located in the NVMe controller's **PCIe MMIO register space (BAR0/BAR1)**. The host reads CAP during controller initialization to determine what the controller supports before enabling it.

# Where CAP is Located

After PCIe enumeration:

1. Host discovers NVMe PCIe device
2. Host maps BAR0/BAR1 into system memory
3. Host reads NVMe registers
4. First register read is usually **CAP**

```
BAR0/BAR1 MMIO Space

Offset
0x0000  CAP      (64 bits)
0x0008  VS       (32 bits)
0x000C  INTMS
0x0010  INTMC
0x0014  CC
0x001C  CSTS
...
```

# CAP Register Layout

The CAP register occupies 64 bits.

```
63                                                         0
+----+----+----+----+----+----+----+----+----+----+----+
|CRMS|MPSM| CPS| BP | PMRS|CMBS| NSSR| DSTRD | TO | MQES|
+----+----+----+----+----+----+----+----+----+----+----+
```

More precisely:

|Bits|Field|Description|
|---|---|---|
|15:0|MQES|Maximum Queue Entries Supported|
|23:16|CQR|Contiguous Queues Required|
|31:24|AMS|Arbitration Mechanisms Supported|
|35:32|Reserved||
|36|TO|Timeout|
|44:37|DSTRD|Doorbell Stride|
|45|NSSRS|NVM Subsystem Reset Supported|
|46|CSS|Command Sets Supported|
|47|BPS|Boot Partition Support|
|51:48|CPS|Controller Power Scope|
|52|MPSMIN|Minimum Page Size|
|55:53|MPSMAX|Maximum Page Size|
|56|PMRS|Persistent Memory Region Supported|
|57|CMBS|Controller Memory Buffer Supported|
|58|NSSS|NVM Subsystem Shutdown Supported|
|59|CRWMS|Controller Ready With Media Support|
|63:60|Reserved||

## 1. MQES (Maximum Queue Entries Supported)

Bits:

```
15:0
```

Specifies maximum entries allowed in an SQ or CQ.

Value stored is:

```
Actual Maximum Queue Size = MQES + 1
```

Example:

```
MQES = 0x0FFF4095 + 1 = 4096 entries
```

Host cannot create queues larger than this.

## 2. CQR (Contiguous Queues Required)

Bit:

```
16
```

### CQR = 1

Queue memory must be physically contiguous.

```
SQ
+----+----+----+----+
| E0 | E1 | E2 | E3 |
+----+----+----+----+

Single contiguous memory region
```

Almost all NVMe SSDs set:

```
CQR = 1
```

## 3. AMS (Arbitration Mechanisms Supported)

Bits:

```
18:17
```

Indicates scheduler types supported.

Possible values:

|Value|Meaning|
|---|---|
|00b|Round Robin|
|01b|Weighted Round Robin|
|10b|Vendor Specific|

Example:

```
AMS = 01b
```

Controller supports weighted arbitration.

Useful when multiple I/O queues compete for resources.

## 4. TO (Timeout)

Bits:

```
31:24
```

Defines controller timeout.

Unit:

```
500 ms
```

Example:

```
TO = 30
```

Timeout:

```
30 × 500 ms

= 15000 ms

= 15 seconds
```

Host uses this when waiting for controller state changes.

Example:

Waiting for:

```
CC.EN = 1
```

and

```
CSTS.RDY = 1
```

## 5. DSTRD (Doorbell Stride)

Bits:

```
35:32
```

Defines spacing between doorbell registers.

Formula:

```
Doorbell Size = 2^(2 + DSTRD)
```

Example:

### DSTRD = 0

```
2^(2+0)

= 4 bytes
```

Doorbells:

```
1000h SQ0TDBL
1004h CQ0HDBL
1008h SQ1TDBL
100Ch CQ1HDBL
```

### DSTRD = 1

```
2^(2+1)= 8 bytes
```

Doorbells:

```
1000h SQ0TDBL
1008h CQ0HDBL
1010h SQ1TDBL
1018h CQ1HDBL
```

Host must use this value to calculate doorbell addresses correctly.

## 6. NSSRS (NVM Subsystem Reset Supported)

Bit:

```
36
```

If set:

```
1 = Supports subsystem reset
```

Host may use:

```
NSSR register
```

to reset all controllers in the subsystem.

## 7. CSS (Command Sets Supported)

Bits:

```
44:37 (newer NVMe revisions)
```

Indicates supported command sets.

Examples:

|Bit|Command Set|
|---|---|
|0|NVM Command Set|
|1|Key-Value|
|2|Zoned Namespace|
Most SSDs:

```
NVM = Supported
```


## 8. BPS (Boot Partition Support)

Bit:

```
45
```

Indicates support for NVMe Boot Partitions.

Useful for:

- Embedded systems
- Servers booting from NVMe

## 9. CPS (Controller Power Scope)

Bits:

```
47:46
```

Indicates scope of power management.

Examples:

- Controller only
- Entire subsystem

## 10. MPSMIN / MPSMAX

These are critical.

### MPSMIN

Bits:

```
51:48
```

Minimum host memory page size supported.

Formula:

```
Page Size = 2^(12 + MPSMIN)
```

Example:

```
MPSMIN = 0
```

```
2^(12+0)

= 4096 bytes
```

### MPSMAX

Bits:

```
55:52
```

Maximum supported page size.

Example:

```
MPSMAX = 4
```

```
2^(12+4)

= 64 KB
```

Host chooses page size in CC.MPS.

Must satisfy:

```
MPSMIN ≤ CC.MPS ≤ MPSMAX
```


## 11. PMRS

Persistent Memory Region Supported.

Bit:

```
56
```

Indicates support for PMR.

PMR can be used as:

- Persistent write buffer
- Persistent cache

Rare in consumer SSDs.

## 12. CMBS

Controller Memory Buffer Supported.

Bit:

```
57
```

If set:

```
Controller Memory Buffer available
```

Host can place:

- Submission Queues
- PRP Lists
- Data buffers

inside controller memory.

Common in DRAM-less SSDs.

## Example Real CAP Decode

Suppose:

```
CAP = 0x004011203F00FFFF
```

Extract:

### MQES

```
FFFFh

65535 entries max
```

### TO

```
3Fh

63 × 500ms

31.5 sec
```

### DSTRD

```
0

4-byte stride
```

### MPSMIN

```
0

4 KB pages
```

### MPSMAX

```
4

64 KB pages
```


# Why CAP Matters During Initialization

Host startup sequence:

```
PCIe Enumeration
        |
        v
Read CAP
        |
        +--> Queue limits
        |
        +--> Timeout value
        |
        +--> Page size limits
        |
        +--> Doorbell spacing
        |
        +--> Supported command sets
        |
        v
Program CC
        |
        v
Enable Controller
```


Without CAP, the host would not know:

- How large queues can be
- Which page sizes are valid
- Where doorbells are located
- Which features are supported

Therefore, **CAP is effectively the NVMe controller's capability descriptor and is the first register the host consults when bringing up an NVMe device.**

---

This register is essential for the host to properly configure and interact with the NVMe controller, as it indicates things like: 

- Queue Configuration: Information regarding the maximum number of queues and queue entries (commands per queue) the controller can support, although actual real-world implementations might support fewer.

- Doorbell Stride: The stride (memory offset) used for accessing the submission and completion queue doorbell registers.

- Memory Buffer Support: Indication of whether the controller supports advanced features like the Controller Memory Buffer (CMB).

- Arbitration Mechanisms: Details on the supported arbitration schemes for command processing. 

  
By reading the CAP register, the host driver can determine the necessary configuration settings and understand the operational limits and supported features of the specific NVMe device it is managing. The capabilities are also detailed in the Identify Controller data structure, which provides a more comprehensive set of information.
