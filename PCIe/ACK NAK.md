
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

