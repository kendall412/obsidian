
> The **VS (Version)** register is a **32-bit read-only NVMe controller register** that tells the host which version of the NVMe specification the controller implements. Like [[Controller Capabilities (CAP)]], VS is located in the controller's PCIe MMIO register space and is typically read during controller initialization.


# Location of VS Register

```
NVMe Controller Registers

Offset
0x0000  CAP   (64-bit)
0x0008  VS    (32-bit)
0x000C  INTMS
0x0010  INTMC
0x0014  CC
0x001C  CSTS
...
```

After PCIe enumeration, the host usually reads:

1. CAP (Controller Capabilities)
2. VS (Version)

before configuring the controller.

# Purpose of VS

The VS register allows the host driver to determine:

- Which NVMe specification revision the controller supports
- Which features may be available
- Whether newer commands and log pages can be used
- How to interpret certain data structures

For example:

```
Older Controller
VS = 1.2

New Controller
VS = 2.0
```

The driver may enable different functionality depending on the reported version.

# VS Register Layout

The register is 32 bits:

```
31                    16 15         8 7         0
+----------------------+-------------+-----------+
|      MJR             |     MNR     |    TER    |
+----------------------+-------------+-----------+
```

| Field | Bits  | Description      |
| ----- | ----- | ---------------- |
| MJR   | 31:16 | Major Version    |
| MNR   | 15:8  | Minor Version    |
| TER   | 7:0   | Tertiary Version |

# Version Formula

```
NVMe Version

MJR.MNR.TER
```

Example:

```
MJR = 1
MNR = 4
TER = 0
```

means:

```
NVMe 1.4
```

# Example Decoding

## Example 1

```
VS = 0x00010300
```

Break it apart:

```
MJR = 0x0001 = 1
MNR = 0x03   = 3
TER = 0x00   = 0
```

Result:

```
NVMe 1.3
```


## Example 2

```
VS = 0x00010400
```

```
MJR = 1
MNR = 4
TER = 0
```

Result:

```
NVMe 1.4
```

## Example 3

```
VS = 0x00020000
```

```
MJR = 2
MNR = 0
TER = 0
```

Result:

```
NVMe 2.0
```

# Why the Host Needs VS

The host driver may support multiple NVMe revisions.

Different versions introduce new features:

|Version|Notable Features|
|---|---|
|NVMe 1.0|Original NVMe specification|
|NVMe 1.1|Improved management features|
|NVMe 1.2|Reservations, datasets|
|NVMe 1.3|Directives, sanitize, telemetry|
|NVMe 1.4|Persistent Event Log, Endurance Groups|
|NVMe 2.0|Modular command-set architecture|

The driver checks VS before using these features.

Example:

```
Read VS

VS = 1.2
```

Driver knows:

```
Telemetry Log PageNot Supported
```

because telemetry was introduced later.

# Example During Initialization

```
PCIe Enumeration
       |
       v
Map BAR0/BAR1
       |
       v
Read CAP
       |
       v
Read VS
       |
       +--> Determine NVMe revision
       |
       +--> Select feature set
       |
       +--> Validate commands
       |
       v
Configure Admin Queues
       |
       v
Enable Controller
```

# Reading VS from Linux

Using `nvme` and PCI utilities (bash):

```
lspci -vv
```

or (bash):

```
setpci -s <device> ...
```

A driver may read (C):

```
u32 vs = readl(mmio + NVME_REG_VS);
```

Then decode (C):

```
major = vs >> 16;
minor = (vs >> 8) & 0xFF;
ter   = vs & 0xFF;
```


# Real Example

Suppose the controller reports:

```
VS = 0x00010400
```

Decode:

```
Major = 1
Minor = 4
Tertiary = 0
```

Result:

```
NVMe Revision 1.4
```

This tells the host that features introduced up through NVMe 1.4 may be available (subject to capability bits reported elsewhere, such as in Identify Controller data).


## Summary

The **VS (Version) register** is a 32-bit read-only register at MMIO offset **0x08** that reports the NVMe specification version implemented by the controller.

```
31                    16 15         8 7         0
+----------------------+-------------+-----------+
|      Major           |   Minor     | Tertiary  |
+----------------------+-------------+-----------+
```

Examples:

|VS Value|Version|
|---|---|
|0x00010000|NVMe 1.0|
|0x00010300|NVMe 1.3|
|0x00010400|NVMe 1.4|
|0x00020000|NVMe 2.0|

The host reads VS during initialization to determine which NVMe features, commands, and data structures are supported by the controller.