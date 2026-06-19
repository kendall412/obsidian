
The **PCIe Data Link Layer (DLL)** sits between the Transaction Layer and the Physical Layer.

Its primary job is:

> **Ensure reliable delivery of TLPs across a single PCIe link.**

> The Transaction Layer creates TLPs. The Data Link Layer makes sure those TLPs arrive correctly at the device on the other end of the link.

## PCIe Stack

```
+-------------------------+
| Transaction Layer       |
|  TLPs                   |
+-------------------------+
| Data Link Layer         |
|  Reliability            |
+-------------------------+
| Physical Layer          |
|  Bits on lanes          |
+-------------------------+
```

## What Does the Data Link Layer Do?

The Data Link Layer is responsible for:

1. Sequence Numbers
2. LCRC Generation
3. [[ACK NAK]] Handling
4. Replay Buffer
5. Flow Control
6. TLP Delivery Verification

It does **not** understand NVMe commands.

It only sees:

```
TLPs
```

from the Transaction Layer.

## Main Idea

Suppose the host sends:

```
Memory Write TLP
```

to an NVMe SSD.

Transaction Layer:

```
Creates TLP
```

Data Link Layer:

```
Adds reliability information
```

Physical Layer:

```
Transmits bits
```

## Data Link Layer Packet Format

The Transaction Layer generates:

```
+------------+
| TLP Header |
+------------+
| Payload    |
+------------+
```

The Data Link Layer adds:

```
+------------------+
| Sequence Number  |
+------------------+
| TLP              |
+------------------+
| LCRC             |
+------------------+
```

## 1. Sequence Number

Every transmitted TLP gets a sequence number.

Example:

```
TLP #100
TLP #101
TLP #102
```

These numbers are used for:

```
ACK
NAK
Replay
```

tracking.

### Why Sequence Numbers?

Suppose:

```
TLP #101
```

gets corrupted.

The receiver can tell the sender:

```
I never received #101 correctly
```

and the sender can retransmit it.

## 2. LCRC (Link CRC)

LCRC stands for:

```
Link Cyclic Redundancy Check
```

The sender computes a CRC over the TLP.

Example:

```
TLP Data
     |
     v
CRC Generator
     |
     v
LCRC
```

The receiver recomputes the CRC.

If:

```
Calculated LCRC
==
Received LCRC
```

the packet is good.

Otherwise:

```
Packet Corrupted
```

### Example LCRC Failure

Sender:

```
Memory Write TLP
LCRC = ABCD
```

During transmission:

```
1 bit flips
```

Receiver computes:

```
LCRC = EFGH
```

Mismatch:

```
ABCD != EFGH
```

Result:

```
Packet rejected
```

## 3. ACK DLLP

If the packet is good:

```
LCRC PASS
Sequence OK
```

receiver sends:

```
ACK DLLP
```

Example:

```
Host -----------------> SSD
     TLP #101

Host <---------------- SSD
        ACK
```

The sender now knows:

```
TLP #101 safely received
```

## 4. NAK DLLP

If packet corruption occurs:

```
LCRC FAIL
```

receiver sends:

```
NAK DLLP
```

Example:

```
Host -----------------> SSD
     TLP #101

Host <---------------- SSD
        NAK
```

Meaning:

```
Please retransmit
```

## 5. Replay Buffer

The sender stores transmitted TLPs.

```
Replay Buffer#100#101#102#103
```

Until ACKed.

### Why?

Suppose:

```
NAK for #101
```

arrives.

Sender retrieves:

```
#101
```

from replay buffer and retransmits.

## Replay Example

```
Host --> #100
Host --> #101
Host --> #102
```

Receiver:

```
#100 OK
#101 Corrupt
#102 Ignored
```

Receiver:

```
NAK #101
```

Host replays:

```
#101
#102
```

from replay buffer.

## 6. Flow Control

PCIe prevents receiver buffer overflow. Receiver advertises credits.

Example:

```
Posted Header Credit
Posted Data Credit
Completion Credit
```

### Credit-Based Flow Control

Receiver:

```
I have space for 8 packets
```

Sender:

```
Can transmit 8 packets
```

After buffer fills:

```
Stop transmitting
```

until more credits arrive.

## DLLPs

The Data Link Layer sends special packets called:

```
DLLPs
```

(Data Link Layer Packets)

Examples:

```
ACK
NAK
Flow Control Update
Power Management
```

## TLP vs DLLP

### TLP

Carries actual transaction.

```
Memory ReadMemory WriteCompletion
```

### DLLP

Carries link management.

```
ACK
NAK
Credits
```

## Example: NVMe Doorbell Write

Host updates:

```
SQ Tail Doorbell
```

Transaction Layer creates:

```
Memory Write TLP
```

Data Link Layer:

```
Assign Seq# = 100
Generate LCRC
Store in Replay Buffer
```

Transmit:

```
TLP #100
```

SSD:

```
Verify LCRC
```

Success:

```
ACK DLLP
```

Host removes:

```
#100
```

from replay buffer.

## Example: NVMe DMA Read

SSD wants command data from host memory.

SSD sends:

```
Memory Read TLP
```

Data Link Layer:

```
Seq# assigned
LCRC generated
Replay buffer updated
```

Host:

```
ACK DLLP
```

SSD removes packet from replay buffer.

## Data Link Layer Initialization

After LTSSM reaches:

```
L0
```

the Data Link Layer performs:

```
Flow Control Initialization
Credit Exchange
```

before normal traffic starts.

## What Happens if Too Many Errors Occur?

Example:

```
Repeated NAKs
Replay Timeouts
Loss of Synchronization
```

Link may enter:

```
Recovery
```

LTSSM retrains the link.

## Transaction Layer vs Data Link Layer

|Transaction Layer|Data Link Layer|
|---|---|
|Creates TLPs|Ensures delivery|
|Memory Read|ACK/NAK|
|Memory Write|Replay|
|Completion|LCRC|
|Configuration|Flow Control|
## Physical Layer vs Data Link Layer

|Physical Layer|Data Link Layer|
|---|---|
|Sends bits|Checks packets|
|Electrical signaling|Reliability|
|Lane alignment|ACK/NAK|
|Equalization|Replay|
## NVMe Example End-to-End

Host rings doorbell:

```
Host Driver
    |
Memory Write (to host memory)
    |
Transaction Layer
    |
Memory Write TLP
    |
Data Link Layer
    |
Seq# + LCRC
    |
Physical Layer
    |
PCIe Lanes
    |
SSD
```

SSD:

```
Check LCRC
Send ACK DLLP
```

Host:

```
Remove TLP from Replay Buffer
```

Transaction complete.

# Summary

The **PCIe Data Link Layer** provides **reliable packet delivery across one PCIe link**.

Its major functions are:

```
Sequence Numbers
LCRC Error Detection
ACK DLLPs
NAK DLLPs
Replay Buffer
Credit-Based Flow Control
```

> For an NVMe SSD, every PCIe transaction—doorbell writes, register reads, DMA reads, DMA writes—passes through the Data Link Layer, which ensures that TLPs are delivered correctly before they reach the Transaction Layer on the other side.

