
> **Admin Queue Attributes (AQA)** is an NVMe controller register that tells the controller the size of the **Admin Submission Queue (ASQ)** and **Admin Completion Queue (ACQ)**. During NVMe initialization, the host programs the AQA register before enabling the controller.

# Purpose of AQA

NVMe uses two special queues for management commands:

1. **Admin Submission Queue (ASQ)**
    - Host places admin commands here.
    - Examples:
        - Identify
        - Get Log Page
        - Create I/O Queue
        - Firmware Download

2. **Admin Completion Queue (ACQ)**
    - Controller places completion entries here.

The controller needs to know how large these queues are. That information is provided through the **AQA register**.

# Register Location

AQA is a memory-mapped register in the NVMe controller register space.

Typical offset:

|Register|Offset|
|---|---|
|CAP|0x00|
|VS|0x08|
|INTMS|0x0C|
|INTMC|0x10|
|CC|0x14|
|CSTS|0x1C|
|**AQA**|**0x24**|
|ASQ|0x28|
|ACQ|0x30|

The host writes AQA before setting `CC.EN = 1`.

# AQA Register Layout

AQA is 32 bits.

```
31                             16 15                              0
+--------------------------------+--------------------------------+
|      ACQS (ACQ Size)           |      ASQS (ASQ Size)           |
+--------------------------------+--------------------------------+
```

## ASQS (Admin Submission Queue Size)

Bits:

```
15:0
```

Stores:

```
Queue Size - 1
```

Formula:

```
ASQ Entries = ASQS + 1
```

Example:

```
ASQS = 63
```

means:

```
ASQ size = 64 entries
```

## ACQS (Admin Completion Queue Size)

Bits:

```
31:16
```

Stores:

```
Queue Size - 1
```

Formula:

```
ACQ Entries = ACQS + 1
```

Example:

```
ACQS = 63
```

means:

```
ACQ size = 64 entries
```

# Example

Suppose host wants:

- ASQ = 64 entries
- ACQ = 64 entries

Then:

```
ASQS = 64 - 1 = 63 = 0x003F
ACQS = 64 - 1 = 63 = 0x003F
```

AQA becomes:

```
31             16 15              0
+----------------+----------------+
| 0x003F         | 0x003F         |
+----------------+----------------+
```

Value written:

```
AQA = 0x003F003F
```

# Why Queue Size − 1?

NVMe stores:

```
N entries → N-1 in register
```

This allows representing:

```
1 entry  -> 064 entry -> 634096 entry -> 4095
```

and is a common hardware convention.

# Relationship with ASQ and ACQ Registers

The AQA register only specifies queue depth.
The actual queue memory locations are specified separately:

### ASQ Register

Contains physical base address of Admin Submission Queue.

Example:

```
ASQ = 0x10000000
```

### ACQ Register

Contains physical base address of Admin Completion Queue.

Example:

```
ACQ = 0x20000000
```

Together:

| Register | Purpose                       |
| -------- | ----------------------------- |
| AQA      | Queue sizes                   |
| ASQ      | Submission queue base address |
| ACQ      | Completion queue base address |

# Controller Initialization Example

Host performs:

### Step 1

Read CAP register.

Determine supported queue sizes.

### Step 2

Allocate memory:

```
ASQ = 64 entries × 64 bytes
     = 4096 bytes

ACQ = 64 entries × 16 bytes
     = 1024 bytes
```

### Step 3

Write AQA:

```
AQA = 0x003F003F
```

### Step 4

Write ASQ base address.

```
ASQ = 0x10000000
```

### Step 5

Write ACQ base address.

```
ACQ = 0x20000000
```

### Step 6

Enable controller:

```
CC.EN = 1
```

### Step 7

Wait until:

```
CSTS.RDY = 1
```

Controller is now ready to accept admin commands.

# Real Example from an NVMe Driver

Linux typically creates:

|Queue|Depth|
|---|---|
|Admin SQ|64|
|Admin CQ|64|

Result:

```
AQA = 0x003F003F
```

Then it sends:

1. Identify Controller
2. Identify Namespace
3. Create I/O Completion Queue
4. Create I/O Submission Queue

using the Admin Queue pair.

# Summary

**Admin Queue Attributes (AQA)** is the NVMe register that defines the depth of the Admin Submission Queue and Admin Completion Queue.

**Bit layout:**

```
31:16 ACQS = ACQ size - 1
15:0  ASQS = ASQ size - 1
```

**Example:**

```
64-entry ASQ
64-entry ACQ

AQA = 0x003F003F
```

AQA works together with:

- **ASQ** → Admin Submission Queue base address
- **ACQ** → Admin Completion Queue base address
- **CC** → Enable controller
- **CSTS** → Controller ready status

These registers form the core of the NVMe controller initialization sequence.

