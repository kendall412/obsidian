

> The **8b/10b mapping table** is the lookup table that converts every **8-bit input byte** into a **10-bit transmission character**.

It was developed by Albert X. Widmer and Peter A. Franaszek at IBM.

Because there are:

- 256 possible 8-bit data bytes
- Two possible running disparity (RD− and RD+)

the complete table contains **hundreds of entries**. It spans several pages in standards documents. Instead of one giant table, the encoder is divided into two smaller lookup tables:

```
8-bit Data Byte
      |
      |
 +------------+
 | 5b / 6b    |
 +------------+
      |
 +------------+
 | 3b / 4b    |
 +------------+
      |
      |
 10-bit Symbol
```

The lower 5 bits are encoded into 6 bits, and the upper 3 bits are encoded into 4 bits.

## 5b/6b Mapping (Examples)

|5-bit Input|Symbol Name|RD− Output|RD+ Output|
|---|---|---|---|
|00000|D.0|100111|011000|
|00001|D.1|011101|100010|
|00010|D.2|101101|010010|
|00011|D.3|110001|110001|
|00100|D.4|110101|001010|
|00101|D.5|101001|101001|
|00110|D.6|011001|011001|
|00111|D.7|111000|000111|

Notice that some entries have **two possible encodings** (one for RD− and one for RD+), while others have only one because they are already balanced.

## 3b/4b Mapping (Examples)

|3-bit Input|Symbol Name|RD− Output|RD+ Output|
|---|---|---|---|
|000|x.0|1011|0100|
|001|x.1|1001|1001|
|010|x.2|0101|0101|
|011|x.3|1100|0011|
|100|x.4|1101|0010|
|101|x.5|1010|1010|
|110|x.6|0110|0110|
|111|x.P7 / x.A7*|Special handling|Special handling|

*The `.7` entries have additional rules because they can create long runs of identical bits, so the encoder chooses among alternate encodings depending on context and running disparity.

### Example: Encoding D21.5

Suppose the data byte is:

```
Binary: 10110101
```

Split into:

```
Upper 3 bits: 101
Lower 5 bits: 10101
```

Encode separately:

```
10101  --> 6-bit code
101    --> 4-bit code
```

Concatenate:

```
6-bit code + 4-bit code
       =
10-bit transmission character
```

The exact 10-bit output depends on the current **running disparity**.

### Running Disparity Example

Suppose the encoder's running disparity is positive:

```
Current RD = +
```

For an input byte, the encoder may choose:

```
RD+ version
```

If the running disparity is negative:

```
Current RD = -
```

it chooses:

```
RD- version
```

This keeps the transmitted stream approximately DC-balanced.

## Special Control Characters (K-Codes)

Besides normal data characters (`D.x.y`), there are **control characters** (`K.x.y`).
Some important examples are:

|Control Character|Common Use|
|---|---|
|K28.5|Comma character, alignment|
|K28.0|Control symbol|
|K23.7|Special protocol control|
|K27.7|Special protocol control|
|K29.7|Special protocol control|
|K30.7|Special protocol control|

For PCIe Gen1 and Gen2, **K28.5** is especially important because it is used for comma detection and receiver alignment during link training.

## Why the Full Table Is Rarely Memorized

Hardware does **not** calculate the mapping mathematically during operation. Instead, the PCIe PHY contains:

```
           8-bit Input
                |
                v
      +-------------------+
      | Lookup Table ROM  |
      +-------------------+
                |
      Running Disparity
                |
                v
         10-bit Output
```

The encoder simply performs a table lookup based on:

1. The 8-bit input value.
2. The current running disparity.

## Summary

The complete 8b/10b mapping table contains **256 data-byte mappings plus control-character mappings**, with separate outputs depending on the running disparity. In practice, it is implemented as two lookup tables:

- **5b → 6b** encoder (32 entries)
- **3b → 4b** encoder (8 entries)

These are combined to produce the final 10-bit transmission character used by PCIe Gen1/Gen2, SATA, Fibre Channel, and other serial protocols.