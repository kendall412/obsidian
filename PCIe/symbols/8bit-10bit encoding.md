
8b/10b encoding is a **line coding scheme** used in high-speed serial communication links to convert every 8 bits of data into a 10-bit transmission symbol.

It was widely used in interfaces such as:

- PCI Express
- Serial ATA
- USB
- Fibre Channel
- Gigabit Ethernet

# Core Idea

The encoder takes:

- 8-bit input byte
- converts it into
- 10-bit code group (symbol)

Example:

```
8-bit data:   10101100
10-bit code:  1001110100
```

This adds **2 extra bits of overhead**.

So efficiency is:

8/10=80%\

meaning 20% bandwidth overhead.

# Why 8b/10b Exists

Raw binary data is problematic on serial links because:

1. Long runs of 0s or 1s make clock recovery difficult
2. DC imbalance causes baseline wander and signal integrity issues
3. Special control symbols are needed for link management
4. Receivers need enough signal transitions to stay synchronized

8b/10b solves all of these.

# Main Features

## 1. DC Balance

The encoding tries to keep:

```
number of 1s ≈ number of 0s
```

over time.

This prevents a DC bias from building up on the channel.

# Running Disparity

A key concept is **running disparity (RD)**.

Running disparity tracks whether more 1s or more 0s have been transmitted.

- Positive disparity → too many 1s
- Negative disparity → too many 0s

The encoder chooses alternate 10-bit symbols to maintain balance.


xample:

A byte may have:

```
RD- version: 1001110100
RD+ version: 0110001011
```

The encoder picks whichever helps rebalance the stream.

## 2. Guaranteed Transitions

8b/10b guarantees enough bit transitions for:

- clock-data recovery (CDR)
- PLL synchronization

Without transitions, the receiver loses timing.

Example bad raw stream:

```
0000000000000000
```

No edges → receiver clock drifts.

8b/10b avoids long runs like this.

Typically maximum consecutive identical bits ≤ 5.


## 3. Special Control Characters

Not all 10-bit symbols represent data.

Some are reserved as **K-characters** (control symbols).

Examples:

|Symbol|Purpose|
|---|---|
|K28.5|Comma / alignment|
|SKP|Clock compensation|
|IDLE|Idle link state|
|STP|Start packet|
|END|End packet|

# Comma Characters

A famous feature:

## K28.5

Contains a unique bit pattern:

```
0011111010
or
1100000101
```

that cannot appear elsewhere.

Receiver uses it for:

- byte alignment
- lane synchronization
- link training

Very important in:

- PCIe Gen1/2
- SATA
- Fibre Channel


# Internal Structure

8b/10b encoding splits the byte:

```
8-bit input:
abcdefgh

split into:

abcde   fgh
(5b)    (3b)
```

Then encoded separately as:

```
5b/6b encoder
3b/4b encoder
```

Result:

```
6 bits + 4 bits = 10 bits
```


# Example Flow

## Transmitter

```
Host Data
   ↓
8b/10b Encoder
   ↓
Serializer
   ↓
Differential PHY
   ↓
Cable / PCB trace
```


## Receiver

```
Differential Signal
   ↓
Deserializer
   ↓
Clock Recovery
   ↓
10b Symbol Detection
   ↓
8b/10b Decoder
   ↓
Recovered Byte
```


# Example in PCIe Gen1/Gen2

PCIe Gen1 rate:

```
2.5 GT/s
```

But because of 8b/10b overhead:

```
Effective payload:
2.0 Gbps
```

Calculation:

2.5×810=2.02.5\times\frac{8}{10}=2.02.5×108​=2.0

PCIe Gen2:

```
5.0 GT/s raw
4.0 Gbps effective
```

# Why PCIe Switched Away From 8b/10b

Starting with:

- PCI Express

PCIe moved to:

## 128b/130b encoding

because 8b/10b wastes too much bandwidth.

Comparison:

|Encoding|Efficiency|
|---|---|
|8b/10b|80%|
|128b/130b|98.46%|

# Error Detection Capability

8b/10b also provides limited error detection.

Invalid symbols can be detected if:

- disparity rules violated
- illegal code groups appear
- unexpected control symbols received

This helps detect physical layer corruption.


# Simple Analogy

Think of 8b/10b as:

> “Packaging data into transmission-safe symbols.”

The extra 2 bits are not payload:

- they help maintain electrical balance
- preserve timing recovery
- embed control information

# Summary

8b/10b encoding:

|Function|Purpose|
|---|---|
|DC balance|Prevent baseline drift|
|Transition density|Enable clock recovery|
|Control symbols|Support protocol management|
|Comma patterns|Allow synchronization|
|Limited error detection|Detect invalid symbols|

# Most Important Takeaway

8b/10b is fundamentally a:

> **Physical-layer signal conditioning and framing technique**

that converts arbitrary binary data into a transmission-friendly serial stream.