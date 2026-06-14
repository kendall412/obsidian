
> When an NVMe SSD first powers up and the PCIe link reaches **L0**, the host cannot immediately send Read and Write commands.

First, the host must initialize the NVMe controller using a series of **Admin Commands** sent through the **Admin Submission Queue (ASQ)** and **Admin Completion Queue (ACQ)**.

Think of Admin Commands as:

> "Management and configuration commands that prepare the SSD for normal I/O."

## High-Level Startup Sequence

```
Power On
   |
   v
PCIe LTSSM
   |
   v
L0
   |
   v
PCIe Enumeration
   |
   v
NVMe Controller Initialization
   |
   v
Admin Commands
   |
   v
Create I/O Queues
   |
   v
Read/Write Commands
```

## Step 1: PCIe Enumeration

The OS discovers the NVMe device and maps BAR0.

The host can now access NVMe registers such as:

|Register|Purpose|
|---|---|
|CAP|Controller capabilities|
|VS|NVMe version|
|CC|Controller configuration|
|CSTS|Controller status|
|AQA|Admin queue attributes|
|ASQ|Admin submission queue address|
|ACQ|Admin completion queue address|

## Step 2: Create Admin Queues

The Admin Queue is special because it exists before any I/O queues.

Host allocates memory:

```
Host RAM

Admin SQ
Admin CQ
```

Example:

```
ASQ = 0x10000000
ACQ = 0x20000000
```

Host programs:

```
AQAASQACQ
```

registers.

## Step 3: Enable Controller

Host sets:

```
CC.EN = 1
```

Controller initializes.

When ready:

```
CSTS.RDY = 1
```

Now Admin Commands may be submitted.

## First Admin Commands Typically Sent

The exact order varies by OS, but commonly:

```
Identify Controller
Identify Namespace(s)
Get Features
Get Log Page
Create IO Completion Queue
Create IO Submission Queue
```

### 1. Identify Controller

Opcode:

```
06h
```

Purpose:

```
Tell me who you are
```

Returns a 4096-byte structure containing:

- Vendor ID
- Model Number
- Serial Number
- Firmware Revision
- Maximum Queue Entries
- Supported Features
- Power States

Example:

```
Model:
Samsung PM9A3

FW:
GDC5902Q
```

This is usually the first important Admin command.

### 2. Identify Namespace

Opcode:

```
06h
```

Different CNS value.

Purpose:

```
Tell me about your storage space
```

Returns:

- Namespace size
- LBA format
- Sector size
- Capacity

Example:

```
Namespace Size:
2 TB

LBA Size:
4096 bytes
```

### 3. Get Features

Opcode:

```
0Ah
```

Purpose:

```
Read current controller settings
```

Examples:

- Number of queues
- Interrupt coalescing
- Power management
- Arbitration settings

### 4. Get Log Page

Opcode:

```
02h
```

Purpose:

```
Retrieve diagnostic information
```

Examples:

- SMART/Health Log
- Error Log
- Firmware Slot Log

The OS may read these during startup.

### 5. Create I/O Completion Queue

Opcode:

```
05h
```

Purpose:

```
Create CQ for user I/O
```

Example:

```
CQID = 1
Depth = 1024
```

Controller now knows where to place completions.

### 6. Create I/O Submission Queue

Opcode:

```
01h
```

Purpose:

```
Create SQ for user I/O
```

Example:

```
SQID = 1
Depth = 1024
Associated CQ = 1
```

Now the SSD is ready for Read/Write commands.

## Example Startup Timeline

```
Host                     NVMe SSD
----                     --------

Read CAP ------------->

Program AQA ---------->

Program ASQ ---------->

Program ACQ ---------->

CC.EN=1 -------------->

                  Initialize Controller

<------------- CSTS.RDY=1

Identify ------------->

<------------- Identify Data

Get Features --------->

<------------- Feature Data

Create IO CQ --------->

<------------- Success

Create IO SQ --------->

<------------- Success

Read/Write I/O Begins 
```

## What the SSD Firmware Does

When it receives these Admin Commands:

### Identify

Firmware gathers:

```
Serial Number
Model Number
Capabilities
```

and fills a 4096-byte buffer.

### Create Queue

Firmware:

```
Stores Queue Base Address
Stores Queue Size
Initializes Queue Context
```


### Get Features

Firmware reads internal configuration values and returns them.

## Why Admin Commands Exist

Without Admin Commands:

```
Host doesn't know:
  Capacity
  Sector Size
  Features
  Queue Limits
```

and

```
No I/O queues exist
```

so normal storage operations cannot start.
Admin Commands provide the management layer that bootstraps the SSD.

## Most Important Startup Commands

In practice, the most critical early Admin Commands are:

|Command|Why|
|---|---|
|Identify Controller|Discover SSD capabilities|
|Identify Namespace|Discover storage geometry|
|Create IO CQ|Create completion queue|
|Create IO SQ|Create submission queue|

These four commands are what transform:

```
PCIe Device
```

into:

```
Operational NVMe Storage Device
```

capable of accepting Read and Write commands.
