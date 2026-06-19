
> **Flow Control** in PCIe is a mechanism that prevents one device from sending more TLPs than the receiving device can buffer.

Think of it as:

> "Before I send packets, I must know you have room to receive them."

Without flow control, a fast transmitter could overflow the receiver's buffers and lose data.

## Why Flow Control Is Needed

Suppose a host can send packets very quickly:

```
Host --------------------> NVMe SSD
```

The SSD has limited internal buffers:

```
SSD Receive Buffer

+--------+
| TLP #1 |
| TLP #2 |
+--------+
```

If the host keeps sending indefinitely:

```
TLP #3
TLP #4
TLP #5
...
```

the SSD's buffer could overflow.

PCIe solves this with **credit-based flow control**.

## Credit-Based Flow Control

The receiver advertises how much buffer space it has.

Example:

```
SSD says:

I have room for:
4 headers
8 data payloads
```

These advertised amounts are called **credits**.

The transmitter is only allowed to send packets if it has sufficient credits.

## Flow Control Initialization

After LTSSM reaches L0:

```
Detect
Polling
Configuration
L0
```

**the Data Link Layers exchange flow-control information.**

```
Host <------ Credits ------ SSD
SSD  <------ Credits ------ Host
```

Only after credits are exchanged can normal TLP traffic begin.

## Types of Credits

PCIe separates credits into different categories.

There are six major credit pools:

|Credit Type|Meaning|
|---|---|
|PH|Posted Header|
|PD|Posted Data|
|NPH|Non-Posted Header|
|NPD|Non-Posted Data|
|CplH|Completion Header|
|CplD|Completion Data|
### Posted Transactions

Examples:

```
Memory Write
Message
```

These do not require a Completion.

They consume:

```
PHPD
```

credits.

### Non-Posted Transactions

Examples:

```
Memory Read
Config Read
Config Write
```

These require a Completion.

They consume:

```
NPHNPD
```

credits.

### Completions

Examples:

```
Completion
Completion with Data
```

These consume:

```
CplH
CplD
```

credits.

## Example: Memory Write

Suppose an NVMe doorbell write:

```
Memory Write TLP
```

contains:

```
Header4-byte payload
```

The transmitter must have:

```
1 Posted Header Credit
1 Posted Data Credit
```

available.

If not:

```
Transmission blocked
```

until more credits arrive.

### Example: Memory Read

Host wants to read CAP register:

```
Memory Read TLP
```

This consumes:

```
1 Non-Posted Header Credit
```

The SSD later returns:

```
Completion with Data
```

which consumes:

```
Completion Header Credit
Completion Data Credit
```

on the return path.

## Credit Advertisement

Receiver advertises:

```
PH = 16
PD = 32
NPH = 8
NPD = 0
CplH = 16
CplD = 64
```

Meaning:

```
I have buffer space for those amounts.
```

## Consuming Credits

Suppose:

```
PH = 4
```

Host sends four Memory Write TLPs.

```
PH = 3
PH = 2
PH = 1
PH = 0
```

Now:

```
No Posted Header Credits Remaining
```

Host must stop sending posted packets.

## Returning Credits

When the receiver processes packets and frees buffer space:

```
Buffer Freed
```

it sends a Flow Control DLLP.

Example:

```
PH += 2
```

meaning:

```
You may send two more posted headers.
```

## Flow Control DLLPs

Credit updates are transmitted using DLLPs.

Examples:

```
UpdateFC DLLP
```

These are Data Link Layer packets.

They are not TLPs.

## Real NVMe Example

### Host Rings Doorbell

Host sends:

```
Memory Write TLP
```

Consumes:

```
PH credit
PD credit
```

SSD stores packet in receive buffer.

Later:

```
Packet processed
Buffer freed
```

SSD returns credits.

## SSD DMA Write Example

SSD wants to write data to host memory.

Suppose SSD transfers:

```
128 KB
```

This becomes many:

```
Memory Write TLPs
```

Before sending each TLP:

```
Check available PH/PD credits
```

If credits run out:

```
Pause transmission
```

until host advertises more.

## Why PCIe Uses Credits Instead of ACK-Based Buffering

Imagine waiting for every packet:

```
Send Packet
Wait
Send Packet
Wait
```

Performance would be poor.

Instead:

```
Receiver:
You may send 64 packets.

Transmitter:
Great, I'll pipeline them.
```

This keeps PCIe highly efficient.

## Flow Control vs ACK/NAK

Many people confuse them.

### [[ACK NAK]]

Purpose:

```
Reliability
```

Questions answered:

```
Did the packet arrive correctly?
```

### Flow Control

Purpose:

```
Buffer management
```

Questions answered:

```
Do you have room for another packet?
```

## Simple Analogy

Imagine a warehouse.

### Flow Control

Warehouse manager says:

```
I have 10 empty shelves.
```

Truck can deliver:

```
10 boxes
```

maximum.

### ACK

After a box arrives:

```
Received successfully.
```

or

```
Box damaged.
Send another.
```

## NVMe End-to-End Example

Host submits a Read command:

```
1. Write SQ Entry
2. Ring Doorbell
```

Doorbell write becomes:

```
Memory Write TLP
```

Flow Control:

```
Enough PH/PD credits?
```

If yes:

```
Transmit
```

ACK/NAK:

```
Packet received correctly?
```

If yes:

```
ACK
```

If no:

```
NAK + Replay
```

# Summary

**PCIe Flow Control** is a credit-based mechanism that prevents receiver buffer overflow.

Key concepts:

```
Credits = Available receive buffer space
```

Credit types:

```
PH   Posted Header
PD   Posted Data
NPH  Non-Posted Header
NPD  Non-Posted Data
CplH Completion Header
CplD Completion Data
```

Operation:

```
Receiver advertises credits
        |
        v
Sender consumes credits
        |
        v
Receiver frees buffers
        |
        v
Receiver returns credits
```

> Flow control ensures that a PCIe device—such as an NVMe SSD or a Root Complex—never receives more TLP traffic than it can safely buffer and process.

