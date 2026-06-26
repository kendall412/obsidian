> The **Polling state** is where two PCIe devices begin actual communication after receiver detection succeeds. The goal is to establish a reliable physical connection before negotiating link width and entering normal operation. It is when the [[Root Complex]], [[Repeater]], and the Endpoint will begin transmitting Ordered Sets of data called [[Training Sequences (TS)]] at PCIe Gen 1 speed in order to establish

### LTSSM Context

```
Detect
  |
  v
Polling
  |
  v
Configuration
  |
  v
L0
```

Think of Polling as:

> "We found each other electrically. Now let's prove we can communicate reliably."

### Polling State Breakdown

For PCIe Gen1–Gen5, the major polling substates are:

```
Polling.Active
      |
      v
Polling.Configuration
      |
      v
Configuration
```

### Step 1: Enter Polling.Active

> Moving into _Polling.Active_ both ends start transmitting TS1 [[Ordered Sets]] with the lane and link numbers set to the PAD symbol. The wait to have sent at least 1024 TSs and received 8 (or their inverse), before moving to _Polling.Config_. Here they start transmitting TS2 ordered sets with link and lane set to PAD, having inverted the RX lanes as necessary. The state then waits for transmitting at least 16 TS2s (after receiving one) and receives at least 8.

After Detect finds a receiver:

```
Lane0 -> Receiver Found
Lane1 -> Receiver Found
Lane2 -> Receiver Found
Lane3 -> Receiver Found
```

the LTSSM enters:

```
Polling.Active
```

At this point, both sides begin transmitting **TS1 Ordered Sets** continuously.

### Step 2: Exchange TS1 Ordered Sets

Both devices send TS1s simultaneously.

```
Root Complex                    NVMe SSD

TS1 -------------------------->
                    <---------- TS1

TS1 -------------------------->
                    <---------- TS1

TS1 -------------------------->
                    <---------- TS1
```

A TS1 contains information such as:

```
Link Number
Lane Number
Training Control Information
```

During early Polling.Active, some fields may be set to special values indicating that lane assignment has not yet been established.

### Step 3: Clock Recovery (CDR Lock a.k.a. [[Bit Lock]])

> Achieve [[Bit Lock]] - Bit Lock refers to the crucial process during link training where **the receiver synchronizes its internal clock with the transmitter's clock and locks onto the incoming data stream's timing** to correctly sample and interpret individual bits, ensuring reliable high-speed data transfer between devices

The receiver must recover a clock from the incoming serial stream.

```
Incoming Serial Data
         |
         v
      CDR Block
         |
         +--> Recovered Clock
```

Without clock recovery:

```
TS1 cannot be decoded
```

and the LTSSM remains in Polling.Active or falls back.

### Step 4: Symbol Lock

> Acquire [[Symbol Lock]] or Block Lock - Symbol Lock is a crucial step during link training where **the receiver successfully aligns its clock and data grouping** ([[PCIe/symbols/general]]) with the transmitter, understanding the specific patterns (like TS1/TS2 Ordered Sets) to correctly decode data, following Bit Lock (clock frequency sync), and allowing the link to move to configuration and normal operation (L0 state). After clock recovery, the receiver must determine symbol boundaries.

For Gen1/Gen2:

```
8b/10b symbols
```

For Gen3+:

```
128b/130b blocks
```

The PHY must correctly identify where symbols begin and end.

```
Bit Stream
    |
    v
Symbol Alignment
    |
    v
Valid PCIe Symbols
```

### Step 5: Receive Valid TS1s

Each side verifies:

```
TS1 received correctly
```

This means:

- Proper encoding
- Correct framing
- No major errors
- Stable reception

The PCIe specification requires receiving a sufficient number of consecutive valid TS1 ordered sets before progressing.

Conceptually:

```
Valid TS1
Valid TS1
Valid TS1
Valid TS1
...
```

### Step 6: Lane Synchronization

For multi-lane links (x2, x4, x8, x16):

```
Lane0
Lane1
Lane2
Lane3
```

signals do not arrive at exactly the same time.

Example:

```
Lane0 arrives first
Lane1 arrives later
Lane2 arrives later
Lane3 arrives later
```

The PHY performs:

```
Deskew
Lane Alignment
```

so that all lanes can be treated as a single logical link.

### Step 7: Transition to Polling.Configuration

After sufficient TS1 exchange and stable reception:

```
Polling.Active      |      vPolling.Configuration
```


### Step 8: Exchange TS2 Ordered Sets

Now both sides begin transmitting TS2.

```
Root Complex                    NVMe SSD

TS2 -------------------------->
                    <---------- TS2

TS2 -------------------------->
                    <---------- TS2
```

TS2 indicates:

> "I successfully received TS1 and am ready to proceed."

### Step 9: Verify Stable Link

During Polling.Configuration both sides verify:

```
Clock Recovery = Stable
Symbol Lock    = Stable
TS2 Reception  = Stable
Lane Sync      = Stable
```

If successful:

```
Polling.Configuration
         |
         v
Configuration
```

### Example: x4 NVMe SSD

Consider a PCIe Gen4 x4 NVMe SSD.

### Detect

```
Lane0 Found
Lane1 Found
Lane2 Found
Lane3 Found
```

### Polling.Active

```
Exchange TS1
Recover Clock
Lock Symbols
Align Lanes
```

### Polling.Configuration

```
Exchange TS2
Confirm Stable Link
```

### Configuration

```
Negotiate Width
Assign Lane Numbers
Prepare Link
```

### L0

```
Normal PCIe Traffic
TLPs
NVMe Commands
DMA Transfers
```

# What Can Go Wrong?

## Stuck in Polling.Active

```
Detect
  |
  v
Polling.Active
```

but never progresses.

Common causes:

- Clock recovery failure
- Corrupted TS1
- Signal integrity issues
- Bad connector
- PHY configuration problems

## Stuck in Polling.Configuration

```
Polling.Active
      |
      v
Polling.Configuration
```

but never reaches Configuration.

Common causes:

- TS2 not being received correctly
- Lane alignment problems
- Equalization issues
- PHY state machine bugs

## Summary

The Polling process consists of:

```
Polling.Active
    |
    |-- Transmit TS1
    |-- Receive TS1
    |-- Clock Recovery
    |-- Symbol Lock
    |-- Lane Synchronization
    |
    v
Polling.Configuration
    |
    |-- Transmit TS2
    |-- Receive TS2
    |-- Verify Stable Link
    |
    v
Configuration
```

The key purpose of Polling is to transform a simple electrical connection discovered in Detect into a stable, synchronized PCIe communication channel ready for link-width negotiation and normal operation.


