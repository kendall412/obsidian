
**128b/130b encoding** is the line encoding scheme used by:

- PCIe Gen3 (8.0 GT/s)
- PCIe Gen4 (16.0 GT/s)
- PCIe Gen5 (32.0 GT/s)

It replaced **8b/10b encoding** because it has much lower overhead while still providing enough information for synchronization.

## Why PCIe Changed from 8b/10b

With 8b/10b:

```
8 bits  → 10 transmitted bits
```

Efficiency:

That means **20% of the bandwidth is used by the encoding itself**. For PCIe Gen3 and beyond, that overhead became too expensive.

## Basic Idea of 128b/130b

Instead of encoding every byte individually:

```
8 bits
↓
10 bits
```

PCIe collects:

```
128 bits
```

and adds only:

```
2 synchronization bits
```

Result:

```
128 bits
   +
2 bits
------
130 transmitted bits
```

Efficiency: Only about **1.54% overhead**.

## High-Level Transmission

Suppose the Transaction Layer creates a stream of TLP bytes.

```
TLP Bytes
      |
      v
128-bit Block
      |
      +-- Add 2 Sync Bits
      |
      v
130-bit Block
      |
      v
Scrambler
      |
      v
Serializer
      |
      v
PCIe Lane
```

## Structure of a 130-bit Block

```
+----------+------------------------+
| Sync Bits| 128-bit Payload        |
| 2 bits   |                        |
+----------+------------------------+
```

The payload may contain:

- TLP data
- DLLP data
- Ordered sets
- Idle information

## The Two Sync Bits

The first two bits indicate what type of block follows.

There are two valid values:

```
01
```

or

```
10
```

Conceptually:

|Sync Bits|Meaning|
|---|---|
|01|Data Block|
|10|Control Block|

(PCIe reserves the other patterns for invalid/error detection.)

### Data Block

```
01+128 bits
```

Contains normal packet data such as:

```
TLP Payload
DLLP Payload
```

### Control Block

```
10
+
128 bits
```

Contains:

- Ordered Sets
- Idle information
- Link management symbols

Examples include:

- SKP Ordered Sets
- Electrical Idle Exit information
- Training-related control information

## Why Only Two Sync Bits?

Unlike 8b/10b, PCIe Gen3+ no longer relies on special comma characters (such as K28.5) for symbol alignment.

Instead:

- The receiver performs **bit lock**
- Then finds **130-bit block boundaries**
- Then checks the 2-bit sync header

If the sync header isn't valid, the receiver knows it has lost alignment.

## Scrambling

Unlike 8b/10b, **128b/130b does not guarantee transition density by itself**.

Instead, PCIe uses a **scrambler**.

Without scrambling:

```
000000000000000000000...
```

would still have no transitions.

The scrambler converts it into something that looks random:

```
10110001101011010010...
```

This provides:

- Frequent transitions for clock recovery
- Better EMI characteristics
- Avoidance of long runs of identical bits

The receiver uses the same scrambling algorithm to recover the original data.

### Example

Original payload:

```
128 bits

101100...
```

Add sync bits:

```
01101100...
```

Scramble:

```
010111001101...
```

Serialize:

```
010111...
```

Transmit on the PCIe lane.

## Receiver Operation

The receiver performs:

```
Receive Serial Bits
        |
        v
Clock Data Recovery (CDR)
        |
        v
Find 130-bit Boundary
        |
        v
Read Sync Bits
        |
        v
Descramble Payload
        |
        v
Recover Original 128 Bits
```

## Why Scrambling Instead of 8b/10b?

8b/10b guarantees:

- DC balance
- Frequent transitions

But costs 20% bandwidth.

128b/130b instead uses:

- Scrambling
- Sync headers

Result:

- Similar signal quality
- Much better efficiency

## PCIe Gen3 Through Gen5

All of these use:

```
128b/130b
```

|Generation|Raw Rate|Encoding|
|---|---|---|
|Gen3|8.0 GT/s|128b/130b|
|Gen4|16.0 GT/s|128b/130b|
|Gen5|32.0 GT/s|128b/130b|
## Effective Bandwidth Example

### PCIe Gen3

Raw signaling:

```
8.0 GT/s
```

Efficiency:

98.46%

Effective data rate per lane:

This is a significant improvement over the 80% efficiency of 8b/10b.

## Ordered Sets

Even with 128b/130b, PCIe still uses ordered sets for:

- Link training
- Lane alignment
- SKP insertion
- Recovery

However, they are carried inside **control blocks**, not encoded as special 10-bit K-codes like in Gen1/Gen2.

## Comparison: 8b/10b vs 128b/130b

|Feature|8b/10b|128b/130b|
|---|---|---|
|Used In|Gen1, Gen2|Gen3, Gen4, Gen5|
|Block Size|8 → 10 bits|128 → 130 bits|
|Overhead|20%|1.54%|
|Clock Recovery|Guaranteed transitions|Scrambler + CDR|
|DC Balance|Running disparity|Statistical via scrambling|
|Control Symbols|K-codes|Control blocks with sync headers|
|Comma Alignment|K28.5|Sync header/block alignment|
## Relation to LTSSM

During Gen3+ link training:

```
Detect
   |
Polling
   |
Configuration
```

the PHY transmits ordered sets inside **128b/130b control blocks**.

The receiver:

```
Bit Lock
     |
Block Lock (130-bit boundary)
     |
Sync Header Check
     |
Descramble
     |
Recognize Ordered Sets
```

Only after reliable block synchronization is achieved can the LTSSM continue toward L0.

# Summary

**128b/130b encoding** is the line encoding method used by PCIe Gen3 through Gen5. It packages **128 bits of payload with a 2-bit synchronization header**, achieving **98.46% efficiency**.

Its key features are:

- **2-bit sync header** identifies data or control blocks.
- **Scrambling** replaces 8b/10b's running disparity to maintain transition density and reduce EMI.
- **130-bit block synchronization** replaces comma alignment.
- **Much lower overhead** (1.54% vs. 20%) enables the much higher bandwidths required for Gen3, Gen4, and Gen5 PCIe links.

