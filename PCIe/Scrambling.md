
> **Scrambling** in PCIe is a process that **pseudo-randomizes the transmitted bit stream** before it is sent over the physical link. The receiver performs the reverse operation (**descrambling**) to recover the original data.

The key idea is:

> **Scrambling changes the transmitted bit pattern, but it does not change the actual information being sent.**

## Why Scrambling Is Needed

Imagine transmitting this data directly:

```
00000000000000000000000000000000
```

or

```
11111111111111111111111111111111
```

These long runs of identical bits create problems:

- Very few signal transitions
- Harder for the receiver's Clock Data Recovery (CDR) circuit
- Increased electromagnetic interference (EMI)
- Spectral peaks that can violate EMC regulations

Instead, PCIe scrambles the data so it looks random.

Example:

Original:

```
00000000000000000000
```

Scrambled:

```
10110100101101001011
```

Notice there are now many transitions.

## Where Scrambling Occurs

For PCIe Gen3 through Gen5:

```
Transaction Layer
        |
        v
      TLP
        |
        v
Data Link Layer
        |
        v
Physical Layer
        |
        +--> Scrambler
        |
        v
Serializer
        |
        v
PCIe Lane
```

On the receive side:

```
PCIe Lane
      |
      v
Deserializer
      |
      v
Descrambler
      |
      v
Original Data
```

## Does Scrambling Encrypt the Data?

No.

It is **not encryption**.

Anyone who knows the scrambling algorithm can immediately recover the original data.

Purpose:

- Improve signal quality
- Improve clock recovery
- Reduce EMI

Not:

- Security
- Privacy

## How Scrambling Works

PCIe uses a **Linear Feedback Shift Register (LFSR)**.

Think of it as a pseudo-random bit generator.

```
          +----------------+
          |     LFSR       |
          +----------------+
                  |
Pseudo-Random Bit Stream
```

Example output:

```
101011001010101110...
```

### XOR Operation

Each transmitted bit is XORed with the pseudo-random bit.

Example:

Original:

```
11001010
```

Pseudo-random sequence:

```
10110111
```

XOR:

```
11001010
10110111
--------
01111101
```

The transmitted data becomes:

```
01111101
```

### Receiver

Receiver generates the exact same pseudo-random sequence.

Received:

```
01111101
```

Pseudo-random sequence:

```
10110111
```

XOR again:

```
01111101
10110111
--------
11001010
```

Original data is recovered.

This works because:

```
A XOR B XOR B = A
```

## Why an LFSR?

An LFSR has several advantages:

- Simple hardware
- Very fast
- Minimal logic
- Long pseudo-random sequence
- Deterministic

Perfect for high-speed PCIe PHYs.

## PCIe Gen1/Gen2 vs Gen3+

### Gen1 / Gen2

Used:

```
8b/10b encoding
```

8b/10b already provides:

- Frequent transitions
- Running disparity
- DC balance

Only limited scrambling is required.

### Gen3 / Gen4 / Gen5

Use:

```
128b/130b encoding
```

128b/130b alone does **not** guarantee enough transitions. Therefore PCIe relies heavily on scrambling.

#### Example

Suppose a TLP contains:

```
AAAAAAAAAAAAAAAA
```

Binary:

```
10101010
10101010
10101010
......
```

Without scrambling:

```
1010101010101010...
```

Strong repeating pattern.

With scrambling:

```
1100010111010010...
```

Looks random.

## Why Random-Looking Data Helps

Random data has:

- More evenly distributed transitions
- Better clock recovery
- Reduced EMI
- Better channel performance

Even though the payload itself may contain long repetitive patterns.

## Scrambling and LTSSM

During link training:

```
Detect
   |
Polling
   |
Configuration
```

Some ordered sets are transmitted **without scrambling** so that both ends can synchronize correctly. Once the link reaches normal operation (L0), ordinary data TLPs are scrambled.

#### Example: NVMe Read

Suppose the SSD DMA-writes user data:

Original:

```
Host Memory Data

00000000
FFFFFFFF
00000000
FFFFFFFF
```

Transaction Layer:

```
Creates Memory Write TLPs
```

Physical Layer:

```
Scramble
```

Transmitted:

```
011010101101...
```

Receiver:

```
Descramble
```

Host memory receives:

```
00000000
FFFFFFFF
00000000
FFFFFFFF
```

exactly as sent.

## Scrambling vs Encoding

These are different operations.

### Encoding

Changes the representation.

Example:

```
8 bits
↓

10 bits
```

or

```
128 bits
↓

130 bits
```

Purpose:

- Framing
- Synchronization

### Scrambling

Keeps the same number of bits.

Example:

```
128 bits
↓

128 scrambled bits
```

Purpose:

- Randomize bit patterns
- Improve signal quality

## Comparison

|Feature|Encoding|Scrambling|
|---|---|---|
|Changes bit count|Yes|No|
|Adds overhead|Yes|No|
|Improves synchronization|Yes|Indirectly|
|Improves EMI|Partly|Yes|
|Makes data look random|No|Yes|
## Why PCIe Uses Both

For Gen4:

```
128 bits
      |
      v
Add 2 Sync Bits
      |
130-bit Block
      |
Scramble Payload
      |
Serialize
      |
Transmit
```

Encoding provides:

- Block boundaries
- Data/control indication

Scrambling provides:

- Transition density
- Good spectral characteristics

# Summary

**Scrambling** is a physical-layer technique that pseudo-randomizes transmitted data using an LFSR.

Process:

```
Original Data
      |
      v
LFSR Generates Pseudo-Random Bits
      |
      v
XOR
      |
      v
Scrambled Data
      |
      v
Transmit
```

Receiver:

```
Received Data
      |
      v
Same LFSR
      |
      v
XOR
      |
      v
Original Data
```

For PCIe:

- **Gen1/Gen2** rely primarily on **8b/10b encoding** for transition density and DC balance.
- **Gen3/Gen4/Gen5** use **128b/130b encoding** plus **scrambling** to achieve high efficiency while maintaining reliable clock recovery and low EMI. Scrambling is transparent to software and NVMe firmware—the original data is fully restored by the receiver before it is processed.


---

> The basic data unit of PCIe is a byte which will be encoded prior to serialization. Before this encoding, though, the data bytes are scrambled on a per lane basis. This is done with a 16-bit linear feedback shift register or an equivalent. The polynomial used for PCIe 2.0 and earlier is G(x) = x16+x5+x4+x3+1, whilst for 3.0 it is G(x) = x23+x21+x16+x8+x5+x2+1. The scrambling can be disabled during initialization, but this is normally for test and debug purposes.

To keep the scrambling synchronized across multiple lanes the [LSFR](https://www.geeksforgeeks.org/digital-logic/linear-feedback-shift-registers-lfsr/) is reset to 0xffff when a COM symbol is processed (see below). Also, it is not advanced when a SKP symbol is encountered since these may be deleted in some lanes for alignment (more later). The K symbols are not scrambled and also data symbols within a training sequence.

Data scrambling: To improve electrical characteristics of a link, data is scrambled. XORing of data stream with a pattern generator by LFSR (Linear Feedback Shift Reg). On Tx side data is scrambled and on Rx side data is de-scrambled.  
Scrambling is performed by serially XORing the 8bit(D0-D7) character with 16bit(D0-D15) output of LFSR. D15 is XORed with D0 of the data to be processed.  
G(x)=x^16 + x^5 + x^4 + x^3 +1  

Rules of data scrambling:  
·      COM symbols initiate the LFSR.  
·      All special symbols (K codes) are not scrambled.  
·      Scrambled is always enabled in Detect by default and disabled at the End of configuration.  
·      Scrambled is not enabled at loopback mode.  
·      All data symbols(D codes) except those within Ordered sets (TS1, TS2, EIEOS) are scrambled.

![[Pasted image 20260127210602.png]]