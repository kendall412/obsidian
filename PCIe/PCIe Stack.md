
## PCIe Protocol Stack Overview

PCIe is organized as a layered protocol, similar to a network stack. Each layer has a specific responsibility.

```
+------------------------------------+
| Software / Driver                  |
+------------------------------------+
| Transaction Layer (TL)             |
+------------------------------------+
| Data Link Layer (DLL)              |
+------------------------------------+
| Physical Layer (PHY)               |
|  - Logical PHY Sub-layer           |
|  - Electrical PHY Sub-layer        |
+------------------------------------+
| PCIe Lane(s)                       |
+------------------------------------+
```

For an NVMe SSD, the path looks like:

```
Application
    |
Operating System
    |
NVMe Driver
    |
PCIe Transaction Layer (TL)
    |
PCIe Data Link Layer (DL)
    |
PCIe Physical Layer (PHY)
    |
PCIe Cable/Connector/Lanes
    |
SSD Physical Layer
    |
SSD Data Link Layer
    |
SSD Transaction Layer
    |
NVMe Controller
```

# 1. Transaction Layer (TL)

The Transaction Layer is the highest PCIe protocol layer.

Its job is to move information between software and devices.

It creates and processes:

- Memory Reads
- Memory Writes
- Configuration Reads
- Configuration Writes
- I/O Transactions (legacy)
- Messages
- Completions

## Transaction Layer Packets (TLPs)

The Transaction Layer generates TLPs.

Example:

Host writes an NVMe command to Submission Queue memory:

```
CPU
 |
 | Memory Write
 v
Transaction Layer
 |
 v
TLP
```

Example Memory Write TLP:

```
+-----------+
| Header    |
+-----------+
| Data      |
+-----------+
```

Header contains:

```
Address
Length
Requester ID
Tag
Type
Attributes
```

## Common TLP Types

### Memory Read

```
Host ------> SSD
```

Request:

```
Read 4096 bytes from address X
```

### Memory Write

```
Host ------> SSD
```

Request:

```
Write data to address X
```

### Completion

```
SSD ------> Host
```

Response to a Memory Read:

```
Completion with Data
```

## Example NVMe Read Command

Host issues:

```
NVMe Read Command
```

Driver writes command to Submission Queue.

PCIe creates:

```
Memory Write TLP
```

to SSD controller memory.

# 2. Data Link Layer (DLL)

The Data Link Layer ensures reliable packet delivery between two directly connected PCIe devices.

Important:

Transaction Layer assumes:

```
Packet Delivered Correctly
```

Data Link Layer makes this happen.

## Responsibilities

### CRC Protection

Every TLP gets a CRC.

Called:

```
LCRC(Link CRC)
```

Example:

```
TLP | + CRC | vTransmit
```

Receiver recalculates CRC.

If mismatch:

```
Packet Corrupted
```


### Sequence Numbers

Every packet gets:

```
Seq #1
Seq #2
Seq #3
```

Example:

```
Packet #10
Packet #11
Packet #12
```

Used for retransmission.

### ACK / NAK

Receiver sends:

```
ACK
```

if packet is correct.

```
NAK
```

if packet is corrupt.

### Replay Buffer

Sender stores transmitted packets.

```
Replay Buffer
+---------+
| Pkt 10  |
| Pkt 11  |
| Pkt 12  |
+---------+
```

If NAK arrives:

```
Retransmit
```

from replay buffer.

## Data Link Layer Packet (DLLP)

Special packets exchanged between devices.

Examples:

```
ACK DLLP
NAK DLLP
Flow Control DLLP
Power Management DLLP
```

DLLPs never reach software.

# 3. Physical Layer [[PCIe PHY]]

The Physical Layer moves bits across PCIe lanes.

It consists of two major pieces:

```
Physical Layer
|
+-- Logical PHY
|
+-- Electrical PHY
```

# 3A. Logical PHY

Handles protocol-related physical functions.

Examples:

### Lane Management

PCIe links may have:

```
x1
x2
x4
x8
x16
```

Logical PHY manages lanes.

### Link Training [[Link Training and Status State Machine (LTSSM)]]

When power is applied:

```
DetectPollingConfigurationRecoveryL0
```

These LTSSM states are controlled here.

### Ordered Sets

Special training symbols:

```
TS1
TS2
SKP
EIOS
```

Used during:

- Link training
- Equalization
- Power management

### Lane [[Deskew]]

Multiple lanes may arrive at different times.

Example:

```
Lane0 arrives
Lane1 arrives later
Lane2 arrives later
```

Deskew logic aligns them.

# 3B. Electrical PHY

This is the actual analog circuitry.

Contains:

```
SERDES
PLL
CDR
TX Driver
RX Front End
CTLE
DFE
```

## SERDES

Serializer/Deserializer.

Converts:

```
Parallel -> Serial
```

and

```
Serial -> Parallel
```

Example:

```
32-bit Data
 |
 v
SERDES
 |
 v
1-bit High-Speed Stream
```

## PLL

Phase Locked Loop.

Creates high-speed clocks.

```
100 MHz Refclk
      |
      v
PLL
      |
      v
8 GHz Internal Clock
```

(Gen5 example)

## CDR

Clock Data Recovery.

PCIe does not transmit a separate clock.

Receiver extracts clock from data stream.

```
Data Stream
     |
     v
CDR
     |
     +--> Clock
     +--> Data
```

## Equalization

Compensates for signal degradation.

### TX Equalization

```
Pre-cursor
Main
Post-cursor
```

Shapes transmitted signal.

### RX Equalization

Uses:

```
CTLE
DFE
```

to restore signal quality.

# How a PCIe Memory Write Travels Through the Stack

Suppose the NVMe driver rings a doorbell.

### Step 1

Software requests:

```
Write 0x1000 to BAR0+1000h
```

### Step 2

Transaction Layer creates TLP.

```
Memory Write TLP
```

### Step 3

Data Link Layer adds:

```
Sequence Number
LCRC
```

### Step 4

Physical Layer:

```
Serialize
Encode
Transmit
```

### Step 5

PCIe lanes carry bits.

```
TX+ TX-
```

### Step 6

SSD PHY receives:

```
Recover Clock
Equalize
Deserialize
```

### Step 7

Data Link Layer verifies:

```
CRC
Sequence Number
```

### Step 8

Transaction Layer extracts:

```
Memory Write
Address
Payload
```

### Step 9

NVMe controller updates:

```
SQ Tail Doorbell
```

and begins processing commands.

# PCIe Layer Summary

|Layer|Main Responsibility|
|---|---|
|Transaction Layer|Creates requests (Read, Write, Config, Completion)|
|Data Link Layer|Reliable delivery, CRC, ACK/NAK, replay|
|Physical Layer (Logical)|Link training, lane management, ordered sets|
|Physical Layer (Electrical)|SERDES, PLL, CDR, equalization|
|PCIe Lanes|Physical transmission medium|

For NVMe firmware work, the most commonly debugged PCIe stack components are:

1. **Transaction Layer** (doorbells, BAR accesses, TLPs)
2. **Data Link Layer** (replay, CRC, ACK/NAK issues)
3. **LTSSM in the Physical Layer** (link training failures, Gen4/Gen5 equalization problems).


