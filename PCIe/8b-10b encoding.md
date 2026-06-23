> 8b/10b encoding is a line code that maps 8-bit data bytes into 10-bit symbols, crucial for high-speed serial communication standards (like Fibre Channel, PCIe) by ensuring DC balance (equal 1s and 0s on average) and sufficient bit transitions for reliable clock recovery (CDR) at the receiver, preventing signal drift and aiding synchronization, despite its 20% bandwidth overhead. It achieves this by using specific 10-bit codes, including special control (K) characters, chosen based on the running disparity (difference between 1s and 0s) to keep the overall stream balanced. While it incurs a **20% bandwidth overhead**, it is widely used across physical layers like USB 3.0, Gigabit Ethernet, and early PCIe to guarantee hardware stability.

It was widely used in:

- PCIe Gen1 (2.5 GT/s)
- PCIe Gen2 (5.0 GT/s)
- SATA
- Fibre Channel
- USB 3.0 (Gen1)
- Gigabit Ethernet variants

Starting with PCIe Gen3, PCIe switched to **128b/130b encoding** because 8b/10b has too much overhead.

## Why 8b/10b Is Needed

If raw bits were transmitted:

```
0000000000000000000000
```

the receiver would have problems:

### 1. Clock Recovery

> The receiver's CDR (Clock Data Recovery) circuit needs transitions. Receivers extract their clock signal from the data stream. Without encoding, long strings of consecutive 0s or 1s (like 00000000) mean no signal changes occur, which causes receivers to lose track of time. The 10-bit mapping forces enough level transitions for reliable clock extraction.

```
1111111111111111111111
```

has no transitions.

No transitions means:

```
CDR loses timing
```

### 2. DC Balance

>  Transmitting too many 1s or 0s causes an electrical buildup (baseline wander) that distorts signals in AC-coupled circuits. 8b/10b balances the total number of 1s and 0s sent, maintaining an equalized baseline.

Long runs of:

```
11111111111111
```

or

```
00000000000000
```

create a DC offset.

High-speed differential links prefer:

```
Number of 1s ≈ Number of 0s
```

### 3. Special Control Characters

>  Out of the 1,024 possible 10-bit symbols, only 256 are used for data. Special symbols are used to mark the beginning and end of data packets, and the strict rules make it easy to detect invalid symbols caused by transmission errors.

The protocol needs symbols such as:

```
COM
SKP
TS1
TS2
Start
End
```

that are distinguishable from normal data.

### Basic Idea

8b/10b converts:

```
8-bit byte
```

into:

```
10-bit transmission character
```

Example:

```
8-bit data:
10110010

Encoded:
1001110100
```

The actual encoding tables are defined by the standard.

## Encoding Process

An 8-bit byte is split into:

```
5 bits
+
3 bits
```

Example:

```
ABCDEFGH

ABCDE
FGH
```

These are encoded separately.

```
5b -> 6b
3b -> 4b
```

Result:

```
6 bits + 4 bits = 10 bits
```

#### Example

Input:

```
8 bits
```

```
00111110
```

Split:

```
00111110
```

Encode:

```
00111 -> 100111
110   -> 0100
```

Transmit:

```
1001110100
```

## Running Disparity

The key concept in 8b/10b is Running Disparity (RD).

Disparity means:

```
(# of 1s) - (# of 0s)
```

Example:

```
11110000

4 ones
4 zeros

Disparity = 0
```

Another example:

```
11111000

5 ones
3 zeros

Disparity = +2
```

Another:

```
00000111

3 ones
5 zeros

Disparity = -2
```

### Why Running Disparity Matters

The encoder tracks whether more:

```
1s
```

or

```
0s
```

have been transmitted.

If disparity becomes too positive:

```
Too many 1s
```

the encoder selects a symbol version with more zeros.

If disparity becomes too negative:

```
Too many 0s
```

the encoder selects a symbol version with more ones.

Result:

```
Average disparity ≈ 0
```

### Example of Running Disparity

Suppose current:

```
RD = +
```

The encoder chooses:

```
Negative-disparity version
```

to balance the stream.

After transmission:

```
RD becomes -
```

The next symbol may use a positive-disparity version.

## Transition Density

8b/10b guarantees frequent transitions.

Example:

Bad raw stream:

```
111111111111111111
```

No edges.

8b/10b encoded stream:

```
1010011010010110
```

Many edges.

Good for:

```
CDR Lock
Bit Recovery
Clock Recovery
```

## Special Characters (K-Codes)

8b/10b supports special symbols called:

```
K-codes
```

Examples:

```
K28.5
K28.0
K23.7
```

These are not normal data bytes.

### K28.5

Most famous control character.

Pattern:

```
0011111010
```

or

```
1100000101
```

depending on running disparity.

Used for:

```
Alignment
Comma Detection
Training Sequences
```

## PCIe Gen1/Gen2 Usage

PCIe Gen1 and Gen2 transmit:

```
TLPs
DLLPs
Ordered Sets
```

after 8b/10b encoding.

Example:

```
TLP Byte Stream
      |
      v
8b/10b Encoder
      |
      v
Serialized Bits
      |
      v
PCIe Lane
```

## TS1 and TS2

During LTSSM Polling:

```
Polling.Active
```

devices exchange:

```
TS1
TS2
```

training sequences.

These contain:

```
K28.5
Control Symbols
Data Symbols
```

encoded using 8b/10b.

## Overhead

8b/10b converts:

```
8 bits
```

into:

```
10 bits
```

Efficiency:

So only **80%** of the transmitted bits are user data.

Overhead:

```
20%
```

### Example: PCIe Gen1

Raw signaling:

```
2.5 GT/s
```

Because of 8b/10b:

```
2.5 × 0.8
=
2.0 Gbps usable
```

per lane.

### Example: PCIe Gen2

Raw:

```
5.0 GT/s
```

Usable:

```
5.0 × 0.8=4.0 Gbps
```

per lane.

## Why PCIe Gen3 Abandoned 8b/10b

At higher speeds the 20% overhead became expensive.

PCIe Gen3 introduced:

```
128b/130b
```

Encoding efficiency:

Much better than:

```
80%
```

## PCIe Encoding Evolution

|PCIe Generation|Encoding|
|---|---|
|Gen1|8b/10b|
|Gen2|8b/10b|
|Gen3|128b/130b|
|Gen4|128b/130b|
|Gen5|128b/130b|
|Gen6|FLIT + PAM4 FEC|
### Example in LTSSM

During:

```
Detect
   |
Polling
```

the transmitter sends:

```
TS1 Ordered Sets
```

Each byte is:

```
8-bit symbol
```

then:

```
8b/10b encoded
```

then:

```
Serialized
```

onto the PCIe lane.

The receiver:

```
Bit Lock
   |
Symbol Lock
   |
8b/10b Decode
   |
Recognize TS1
```

## Summary

**8b/10b encoding** converts:

```
8 bits of data
```

into:

```
10 transmitted bits
```

to provide:

```
Clock recovery
DC balance
Frequent transitions
Control characters (K-codes)
Comma alignment
```

Key properties:

|Feature|Purpose|
|---|---|
|Running Disparity|Maintain DC balance|
|K-Codes|Special protocol symbols|
|Transition Density|Enable CDR lock|
|20% Overhead|Cost of encoding|

For PCIe:

```
Gen1/Gen2 -> 8b/10b
Gen3+     -> 128b/130b
```

> During PCIe link training (TS1/TS2), ordered sets and control symbols are encoded with 8b/10b on Gen1/Gen2 links before being transmitted across the PCIe lanes.







---
How it works

Splits data: An 8-bit byte is divided into a 5-bit and a 3-bit group.

Maps to codes: These groups are mapped to 6-bit and 4-bit codes, respectively, from pre-defined tables.

Maintains disparity: The selection of codes (e.g., from two possible 10-bit codes for some data) depends on the running disparity, ensuring the total number of 1s and 0s stays balanced over time.

Adds overhead: The 2 extra bits per byte add a 25% overhead, meaning a 1 Gbps data stream needs a 1.25 Gbps physical link.

Includes special characters: It uses special K-characters (e.g., commas like 0011111 and 1100000) for control signals, not just data. 

Key Benefits

DC Balance: Prevents DC offset, crucial for optical links and maintaining signal integrity.

Clock Recovery: Ensures enough 0-to-1 or 1-to-0 transitions for the receiver to lock onto the clock.

Error Detection: Can detect single-bit errors and other transmission faults. 

Common Uses

Fibre Channel

Gigabit Ethernet (1000BASE-X)

PCI Express (PCIe)

Serial Attached SCSI (SAS)

USB 3.0 (SuperSpeed USB)