
> **D-codes** (Data Characters) are the **normal data symbols** used in **8b/10b encoding**. They represent ordinary 8-bit data bytes that are transmitted over the PCIe link. They are called **D-codes** because they carry **Data**, in contrast to **K-codes**, which carry **Control** information. For **PCIe Gen1 and Gen2**, every transmitted data byte is converted into a D-code before it is serialized onto the PCIe lanes.

## D-Code vs K-Code

```
Application Data
       |
       v
8-bit Byte
       |
8b/10b Encoder
       |
       +--> D-Code (normal data)
       |
       +--> K-Code (control symbol)
```

Examples:

|Type|Meaning|
|---|---|
|D21.5|Normal data character|
|D10.2|Normal data character|
|K28.5|Control character (comma)|
|K28.0|Control character|

## What Does D21.5 Mean?

A D-code is written as:

```
Dx.y
```

Example:

```
D21.5
```

The notation comes from how 8b/10b splits an 8-bit byte:

```
8-bit Byte

ABCDEFGH

ABCDE   FGH
```

or numerically:

```
Lower 5 bits
+
Upper 3 bits
```

For D21.5:

```
21 = lower 5 bits

5 = upper 3 bits
```

The encoder independently encodes:

```
21
```

using the **5b→6b table**, and

```
5
```

using the **3b→4b table**.

Then it concatenates the results:

```
6 bits
+
4 bits
=
10 transmitted bits
```

### Example Encoding

Suppose the data byte is:

```
10110101
```

Split:

```
Upper 3 bits

101

Lower 5 bits

10101
```

The encoder performs:

```
10101
    |
    v
5b/6b Lookup

101
    |
    v
3b/4b Lookup
```

Result:

```
10-bit D-Code
```

The exact 10-bit value depends on the current **Running Disparity (RD)**.

## Running Disparity

Many D-codes have **two possible encodings**:

```
RD-
```

and

```
RD+
```

Example (conceptual):

```
D0.0

RD-
1001111011

RD+
0110000100
```

The encoder chooses the appropriate version to keep the transmitted stream DC-balanced.

## D-Codes Carry Everything That Isn't Control

When PCIe sends:

- TLPs
- DLLPs
- Payload data

the bytes are transmitted as D-codes.

For example, if a TLP contains:

```
Byte0 = 0x35
Byte1 = 0x8A
Byte2 = 0x10
```

each byte becomes a D-code:

```
Dxx.y
Dxx.y
Dxx.y
```

before transmission.

## D-Codes in a TLP

Suppose a Memory Write TLP contains:

```
Header

01
02
03
04

Payload

AA
BB
CC
DD
```

Each byte is encoded:

```
01 -> D...
02 -> D...
03 -> D...
04 -> D...

AA -> D...
BB -> D...
```

The receiver decodes the D-codes back into the original bytes.

## D-Codes During Link Training

Even during LTSSM training, ordered sets contain both K-codes and D-codes.

Example TS1 (conceptual):

```
K28.5
D10.2
D10.2
D21.5
D21.5
```

Meaning:

- **K28.5** is the comma/control character.
- **D10.2** and **D21.5** are ordinary data characters carrying fields defined by the PCIe specification, such as lane number, link number, and training information.

## How the Receiver Decodes D-Codes

The receiver performs:

```
Receive Serial Bits
        |
        v
Clock Recovery
        |
        v
Find Symbol Boundary
        |
        v
10-bit Symbol
        |
        v
8b/10b Decoder
        |
        v
8-bit Byte
```

If the 10-bit symbol matches a valid D-code:

```
D21.5
```

it is decoded into its original 8-bit byte.

## D-Codes vs K-Codes

|D-Code|K-Code|
|---|---|
|Represents normal 8-bit data|Represents control information|
|Used in TLPs and DLLPs|Used for protocol control|
|256 possible data bytes|Limited set of reserved control symbols|
|Decoded into user/protocol data|Interpreted as protocol events|

## Example During PCIe Gen2 Link Training

During Polling.Active:

```
TS1 Ordered Set

K28.5
D10.2
D10.2
D21.5
D21.5
...
```

- The **K28.5** helps the receiver establish symbol alignment.
- The following **D-codes** carry the training information defined by the PCIe specification.

## Why D-Codes Disappear in PCIe Gen3+

PCIe Gen3 and later no longer use 8b/10b encoding.

Instead:

```
128 bits
```

are transmitted in:

```
130-bit blocks
```

using **128b/130b encoding**.

Therefore:

- No D-codes
- No K-codes

Instead, Gen3+ uses:

- **Data Blocks**
- **Control Blocks**
- **2-bit Sync Headers**

# Summary

A **D-code (Data Character)** is the **10-bit 8b/10b-encoded representation of a normal 8-bit data byte**.

For PCIe Gen1 and Gen2:

```
8-bit Byte
     |
8b/10b Encoder
     |
10-bit D-Code
     |
Transmit
```

Key points:

- D-codes carry **normal data**.
- They are named as **Dx.y**, where `x` comes from the lower 5 bits and `y` comes from the upper 3 bits of the original byte.
- Most D-codes have two encodings (RD− and RD+) to maintain running disparity.
- They are used throughout TLPs, DLLPs, and ordered sets alongside K-codes.
- **PCIe Gen3 and later do not use D-codes**, because they replaced 8b/10b encoding with 128b/130b encoding.