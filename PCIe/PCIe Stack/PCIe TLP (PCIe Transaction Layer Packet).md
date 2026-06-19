
> A **PCIe TLP (Transaction Layer Packet)** is the fundamental packet used by PCIe devices to communicate. Think of a TLP as the PCIe equivalent of a network packet. For an NVMe SSD, almost every operation eventually becomes one or more PCIe TLPs.

## PCIe Stack Context

```
Application
     |
NVMe Driver
     |
PCIe Transaction Layer (TL)
     |
PCIe Data Link Layer (DLL)
     |
PCIe Physical Layer (PHY)
```

The **Transaction Layer** creates TLPs.

## Why TLPs Exist

Suppose the CPU wants to ring an NVMe doorbell:

```
Write 0x10 to SQ0 Tail Doorbell
```

PCIe converts this into a:

```
Memory Write TLP
```

and sends it to the SSD.

## What Does a TLP Contain?

A TLP generally contains:

```
+----------------+
| Header         |
+----------------+
| Data (optional)|
+----------------+
| Digest(optional)|
+----------------+
```

## Example: Memory Write TLP

```
Host writes:
0x12345678
to
0x80001000
```

The TLP might look conceptually like:

```
+-------------------------+
| Type = Memory Write     |
| Address = 0x80001000    |
| Length = 4 bytes        |
+-------------------------+
| Data = 0x12345678       |
+-------------------------+
```

## Types of PCIe TLPs

PCIe defines several TLP types.

### 1. Memory Read

```
Requester ------> Device
```

Example:

```
Read NVMe CAP Register
```

Host sends:

```
Memory Read TLP
```


### 2. Memory Write

```
Requester ------> Device
```

Example:

```
Ring SQ Doorbell
```

Host sends:

```
Memory Write TLP
```

### 3. Completion

Response to a Memory Read.

```
Host ---- Read Request ---->

SSD  <--- Completion -------
```

Example:

```
CAP Register Value
```

returned in a Completion TLP.

### 4. Configuration Read

Used during PCIe enumeration.

Example:

```
Read Vendor ID
Read Device ID
```

### 5. Configuration Write

Example:

```
Program BAR Registers
```

### 6. Message TLP

Examples:

```
Power Management
Hot Plug
Error Reporting
```

## Example: NVMe Doorbell Write

Suppose:

```
SQ Tail Doorbell Address= 0x80001000
```

Host updates:

```
Tail = 5
```

PCIe generates:

```
Memory Write TLP
```

Conceptually:

```
Type    = Memory Write
Address = 0x80001000
Data    = 5
```

SSD receives the TLP and updates the doorbell register.

## Example: Reading CAP Register

Host wants:

```
CAP Register
```

located at:

```
BAR0 + 0x0
```

### Step 1

Host sends:

```
Memory Read TLP
```

```
Type    = Memory Read
Address = 0x80000000
Length  = 8 bytes
```

### Step 2

SSD responds:

```
Completion TLP
```

```
Data = CAP Register Value
```

## TLP Header Fields

A TLP header contains information such as:

|Field|Purpose|
|---|---|
|Fmt|Format|
|Type|Read, Write, Completion, etc.|
|Length|Payload size|
|Requester ID|Who sent it|
|Tag|Match request/response|
|Address|Target address|
|Attributes|Ordering, priority|
Example:

```
Memory Read
Address = 0x10000000
Length  = 16 DW
Tag     = 0x15
```

## TLP vs DLLP

Many engineers confuse these.

### TLP

Carries actual transactions.

Examples:

```
Memory Read
Memory Write
Completion
Configuration Read
```

### DLLP

Used by the Data Link Layer.

Examples:

```
ACK
NAK
Flow Control
```

DLLPs never reach software.

## TLP Journey Through PCIe

Suppose host rings an NVMe doorbell.

### Transaction Layer

Creates:

```
Memory Write TLP
```

### Data Link Layer

Adds:

```
Sequence Number
LCRC
```

### Physical Layer

Converts packet to serial bits.

```
Serialize
Transmit
```

### SSD

Receives:

```
Deserialize
Verify LCRC
Process TLP
```

## NVMe Examples of TLP Usage

### During Initialization

Host programs:

```
AQAASQACQCC
```

using:

```
Memory Write TLPs
```

## During Read Commands

Host writes command into Submission Queue memory.

Then:

```
Memory Write TLP
```

rings the SQ doorbell.

## During DMA

The SSD performs:

```
Memory Read TLPs
Memory Write TLPs
```

to access host memory.

## Real NVMe Read Example

```
Application
     |
Read File
     |
NVMe Driver
     |
Create Read Command
     |
Write SQ Entry
     |
Ring Doorbell
```

Doorbell operation becomes:

```
Memory Write TLP
```

SSD then fetches the command using:

```
Memory Read TLP
```

and returns data using:

```
Memory Write TLP
```

into host memory.

## Summary

A **PCIe TLP (Transaction Layer Packet)** is the primary packet used by PCIe to perform transactions.

Common TLP types:

|TLP Type|Purpose|
|---|---|
|Memory Read|Read memory/registers|
|Memory Write|Write memory/registers|
|Completion|Return read data|
|Config Read|Enumeration|
|Config Write|Device setup|
|Message|Events and notifications|
For NVMe SSDs, TLPs are used for:

```
Doorbell Writes
Register Accesses
Queue Fetches
DMA Transfers
PCIe Enumeration
```

In short:

> NVMe commands are storage commands, but PCIe TLPs are the transport packets that carry the underlying memory transactions between the host and the SSD.

