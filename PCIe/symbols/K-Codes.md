
> **K-codes** (also called **control characters** or **special symbols**) are special **10-bit transmission characters** in **8b/10b encoding** that **do not represent normal data bytes**. They are used by the protocol itself rather than by software or applications.

Think of it this way:

- **D-codes (Data characters)** = Carry user data.
- **K-codes (Control characters)** = Control the communication link.

### D-Code vs K-Code

Normal data:

```
Application Data
      |
0x35
0xA1
0x7F
```

becomes **D-Codes**:

```
D21.5
D1.7
D31.4
```

---

Control information:

```
Start Link Training
Align Receiver
Idle
Special Marker
```

becomes **K-Codes**:

```
K28.5
K28.0
K27.7
```

## Why Are K-Codes Needed?

A protocol sometimes needs to transmit information that is **not user data**.

Examples:

- Align receiver
- Synchronize lanes
- Start link training
- Indicate idle
- Mark packet boundaries (in some protocols)

Using ordinary data bytes would be ambiguous. Instead, the protocol uses reserved K-codes.

## K-Code Naming

A K-code is written as:

```
Kx.y
```

Example:

```
K28.5
```

where:

- **K** = Control character
- **28** = 5-bit portion
- **5** = 3-bit portion

Similarly,

```
D21.5
```

means:

- **D** = Data character
- **21** = lower 5 bits
- **5** = upper 3 bits

## Most Important K-Code: K28.5

K28.5 is by far the most famous.

It is called the **comma character**.

Depending on running disparity, it is encoded as one of two 10-bit patterns.

Conceptually:

```
RD-0011111010
```

or

```
RD+

1100000101
```

These bit patterns contain a unique sequence that **cannot appear inside any valid data character**. This allows the receiver to determine symbol boundaries.

## Why Is It Called a Comma?

Imagine the receiver is seeing a continuous serial stream:

```
101100111010001011...
```

Initially it does not know where one 10-bit symbol begins.

It searches for the unique K28.5 pattern:

```
........0011111010........
```

Once found:

```
|0011111010|1001110101|...
```

the receiver knows exactly where each 10-bit symbol starts. This process is called **comma detection**.

## K28.5 in PCIe

During PCIe Gen1/Gen2 link training:

```
PERST#
    |
Detect
    |
Polling
```

the transmitter repeatedly sends ordered sets containing:

```
K28.5
```

The receiver:

1. Locks its CDR (Clock Data Recovery).
2. Detects the K28.5 comma.
3. Determines the 10-bit symbol boundaries.
4. Starts decoding 8b/10b characters correctly.

Without K28.5, the receiver would not know where each encoded character begins.

## Ordered Sets

PCIe uses K-codes inside **ordered sets**.

For example:

```
TS1

K28.5
D10.2
D10.2
D21.5
...
```

or

```
TS2

K28.5
D5.2
D5.2
...
```

These ordered sets are exchanged during LTSSM states such as:

```
Polling.Active
Configuration
Recovery
```

## Common K-Codes

Some commonly used control characters include:

|K-Code|Typical Purpose|
|---|---|
|K28.5|Comma alignment, synchronization|
|K28.0|Control symbol|
|K23.7|Special protocol control|
|K27.7|Special protocol control|
|K29.7|Special protocol control|
|K30.7|Special protocol control|

Different protocols use these K-codes differently.

## PCIe vs Other Protocols

PCIe Gen1/Gen2:

```
K28.5
```

used for:

- Receiver alignment
- Ordered sets
- Link training

Other protocols like SATA and Fibre Channel also use K-codes, but sometimes assign them different meanings.

## Hardware Implementation

Inside the PCIe PHY:

```
8-bit Value
      |
      v
8b/10b Encoder
      |
      +--> D-Code
      |
      +--> K-Code
      |
      v
10-bit Symbol
```

The encoder knows whether the input is:

- a normal data byte, or
- a control character requested by the PCIe controller.

### Example During Link Training

During Polling:

```
LTSSM
 |
Polling.Active
```

Transmitter repeatedly sends:

```
TS1

K28.5
D10.2
D10.2
D10.2
...
```

Receiver:

```
Receive Bits
      |
      v
Find K28.5
      |
      v
Determine Symbol Boundary
      |
      v
Decode Remaining Characters
```

## Why K-Codes Disappeared in Gen3+

PCIe Gen3 and later use **128b/130b encoding** instead of 8b/10b.

With 128b/130b:

- There are **no K-codes**.
- Symbol alignment is achieved using the **2-bit sync header** and 130-bit block boundaries.
- Ordered sets are carried inside **control blocks**, not as special 10-bit control characters.

# Summary

A **K-code** is a special **8b/10b control character** reserved for protocol operations rather than user data.

Key points:

- **D-codes** carry data.
- **K-codes** carry protocol control information.
- **K28.5** is the most important K-code because it is the **comma character** used for receiver alignment during PCIe Gen1/Gen2 link training.
- K-codes are fundamental to **8b/10b encoding** and are **not used in PCIe Gen3 and later**, which instead rely on 128b/130b block synchronization.

