
> **ACK** and **NAK** are special PCIe **DLLPs (Data Link Layer Packets)** used by the **PCIe Data Link Layer** to guarantee reliable delivery of TLPs across a PCIe link.

Think of them as:

```
ACK = "I received it correctly."

NAK = "I received it incorrectly.
       Please send it again."
```

## Where ACK/NAK Fit in PCIe

```
Transaction Layer
      |
      v
     TLP
      |
      v
Data Link Layer
      |
      +--> Sequence Number
      +--> LCRC
      |
      v
Physical Layer
```

When a TLP is received, the Data Link Layer decides:

```
Good Packet?  ---> ACK

Bad Packet?   ---> NAK
```

### Example: ACK

Host sends a TLP:

```
Memory Write TLP
Seq# 100
```

```
Host --------------------> SSD
      TLP #100
```

SSD checks:

```
Sequence Number OK
LCRC OK
```

SSD sends:

```
ACK DLLP
```

```
Host <-------------------- SSD
           ACK
```

Meaning:

```
TLP #100 received correctly.
```

The host can now remove TLP #100 from its replay buffer.

### Example: NAK

Host sends:

```
TLP #101
```

During transmission:

```
1 bit corrupted
```

Receiver calculates:

```
Received LCRC != Calculated LCRC
```

The packet is bad.

SSD sends:

```
NAK DLLP
```

```
Host <-------------------- SSD
           NAK
```

Meaning:

```
Please retransmit.
```

The host retrieves the packet from its replay buffer and sends it again.

## Why ACK/NAK Are Needed

PCIe links can experience:

- Electrical noise
- Signal integrity issues
- Crosstalk
- Random bit errors

Even though errors are rare, PCIe must guarantee correct delivery.

Without ACK/NAK:

```
Corrupted doorbell write
Corrupted DMA transfer
Corrupted register access
```

could cause system failures.

## Replay Buffer

Every transmitted TLP is stored temporarily.

Example:

```
Replay Buffer

#100
#101
#102
```

When ACK arrives:

```
ACK through #101
```

the sender can discard:

```
#100
#101
```

from the replay buffer.

## ACK Is Cumulative

PCIe ACKs are not usually sent for each packet individually.

Example:

```
Host sends:

#100
#101
#102
#103
```

Receiver successfully receives all four.

Instead of:

```
ACK #100
ACK #101
ACK #102
ACK #103
```

it may send:

```
ACK #103
```

meaning:

```
Everything through #103 is good.
```

This reduces overhead.

### NAK Example

Suppose:

```
#100 OK
#101 Corrupted
#102 Received
#103 Received
```

Receiver sends:

```
NAK #101
```

Sender retransmits starting from:

```
#101
#102
#103
```

from the replay buffer.

## What Does the Receiver Check?

Before sending ACK:

### 1. LCRC

The receiver verifies the Link CRC.

```
LCRC Match?
```

If yes:

```
Packet valid
```

If no:

```
Packet corrupted
```

### 2. Sequence Number

Receiver checks:

```
Expected #101

Received #101
```

Good.

But if:

```
Expected #101

Received #105
```

something is wrong.

NAK may be generated.

## ACK/NAK Are DLLPs

Important distinction:

### TLP

Transaction Layer Packet

Examples:

```
Memory Read
Memory Write
Completion
```

### DLLP

Data Link Layer Packet

Examples:

```
ACK
NAK
Flow Control Update
```

ACK/NAK never reach NVMe firmware.

They are handled entirely by PCIe hardware.

#### NVMe Doorbell Example

Host updates SQ Tail Doorbell.

Transaction Layer creates:

```
Memory Write TLP
```

Data Link Layer adds:

```
Seq# = 200
LCRC
```

Transmit:

```
Host --------------------> SSD
        TLP #200
```

SSD verifies packet.

If good:

```
SSD --------------------> Host
           ACK
```

Doorbell write is considered successfully delivered.

#### NVMe DMA Example

SSD wants to read a command from host memory.

SSD sends:

```
Memory Read TLP
```

Host checks:

```
LCRC OK
Sequence OK
```

Host sends:

```
ACK DLLP
```

SSD knows the request arrived correctly.

## ACK/NAK and LTSSM Recovery

If many NAKs occur:

```
Repeated Replay
Repeated CRC Errors
Replay Timeout
```

the link may be considered unstable.

The LTSSM can transition:

```
L0
 |
 v
Recovery
```

to retrain the link.

## Real PCIe Gen4 Example

Suppose:

```
PCIe Gen4 x4
```

A noisy lane causes occasional errors.

Sequence:

```
TLP #500 sent
```

Receiver detects:

```
LCRC failure
```

Receiver sends:

```
NAK
```

Sender replays:

```
TLP #500
```

This happens automatically in hardware. Software and NVMe firmware never see it.

## Summary

ACK and NAK are **Data Link Layer Packets (DLLPs)** used by PCIe hardware for reliable delivery.

|DLLP|Meaning|
|---|---|
|ACK|Packet received correctly|
|NAK|Packet corrupted, retransmit|

Flow:

```
Sender
   |
   | TLP
   v
Receiver
   |
   +--> Good → ACK
   |
   +--> Bad  → NAK
```

ogether with:

```
Sequence Numbers
Replay Buffers
LCRC
```

> ACK and NAK allow PCIe to reliably transport TLPs between devices such as a host and an NVMe SSD, even when occasional transmission errors occur.

