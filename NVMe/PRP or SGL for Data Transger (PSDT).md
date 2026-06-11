
**PSDT** stands for **PRP or SGL for Data Transfer**.

> It is a field in **Command Dword 0 (CDW0)** of every NVMe command and tells the controller how the command's data buffer is described.

# Where is PSDT Located?

An NVMe command is 64 bytes.

The first DWORD (CDW0) contains:

```
31                              16 15     14 13   12 11       0
+--------------------------------+---------+-------+----------+
| CID                            | PSDT    | FUSE  | OPC      |
+--------------------------------+---------+-------+----------+
```

Fields:

| Field | Description          |
| ----- | -------------------- |
| OPC   | Opcode               |
| FUSE  | Fused operation info |
| PSDT  | Data pointer type    |
| CID   | Command Identifier   |

# Why PSDT Exists

NVMe commands need to tell the controller where the host data buffer is located.

There are two mechanisms:

1. **PRP (Physical Region Page)**
2. **SGL (Scatter-Gather List)**

The controller looks at PSDT to determine which interpretation to use.

# PSDT Values

For most NVMe versions:

| PSDT Value | Meaning  |
| ---------- | -------- |
| 00b        | PRP used |
| 01b        | SGL used |
| 10b        | Reserved |
| 11b        | Reserved |
## PSDT = 00b (PRP)

Most common case.

The controller interprets:

```
PRP1
PRP2
```

as PRP pointers.

Example:

```
Read Command

PSDT = 00b

PRP1 = 0x10000000
PRP2 = 0x20000000
```

Controller:

```
Use PRP mechanism
```

## PSDT = 01b (SGL)

Controller interprets the data pointer fields as SGL descriptors.

Example:

```
Read Command

PSDT = 01b
```

Now:

```
MPTR / Data Pointer Area
```

contains SGL information rather than PRP information.

Controller:

```
Walk SGL descriptors
```

to locate host memory.

# Example Read Command Using PRP

```
CDW0
-----
OPC  = 0x02 (Read)
PSDT = 00b
CID  = 15

PRP1 = 0x10000000
PRP2 = 0x10001000
```

Controller executes:

```
Use PRP1/PRP2
DMA data
```

# Example Read Command Using SGL

```
CDW0
-----
OPC  = 0x02
PSDT = 01b
CID  = 16
```

Data pointer area contains:

```
SGL Descriptor

Address = 0x30000000
Length  = 65536
Type    = Data Block
```

Controller executes:

```
Use SGL descriptor
DMA data
```

# How Firmware Uses PSDT

When the SSD firmware receives a command:

```
Fetch SQ Entry
       |
       v
Parse CDW0
       |
       v
Read PSDT
```

Decision:

```
PSDT == 00 ?
       |
      Yes
       |
       v
Parse PRP1/PRP2

No
 |
 v
Parse SGL
```

Pseudo-code (C):

```
if (psdt == 0)
{
    process_prp();
}
else if (psdt == 1)
{
    process_sgl();
}
else
{
    return INVALID_FIELD;
}
```

# Relation to Identify Controller

Before using SGL, the host should check whether the controller supports it.

In the Identify Controller data structure there is an SGL support field:

```
SGLS
```

If SGL support is not advertised:

```
Host must use PRP
```

even if it would prefer SGL.

# What Happens in Most Consumer SSDs?

Typical flow:

```
Linux NVMe Driver
      |
      v
Read Command
PSDT = 00
      |
      v
PRP1 / PRP2
```

Most commands you see in consumer systems use:

```
PSDT = 00b
```

meaning PRPs.

# Real Command Example

Suppose a Read command starts with:

```
CDW0 = 0x00170002
```

Breaking it down conceptually:

```
OPC  = 0x02 (Read)
CID  = 0x17
PSDT = 00
```

Controller interprets:

```
Use PRP pointers
```

## Summary

**PSDT (PRP or SGL for Data Transfer)** is a 2-bit field in **CDW0** that tells the NVMe controller how to interpret the command's data buffer pointers.

|PSDT|Meaning|
|---|---|
|00b|PRP data pointers|
|01b|SGL descriptors|
|10b|Reserved|
|11b|Reserved|
Controller processing:

```
Read Command
      |
      v
Check PSDT
      |
      +--> 00 → Use PRP1/PRP2
      |
      +--> 01 → Use SGL descriptors
```

For most NVMe SSDs in PCs and servers, **PSDT is usually `00b` (PRP mode)**, while **SGL mode (`01b`) is more common in enterprise and NVMe-over-Fabrics environments**.

