
> The **NVMe Set Features command** is an **[[Admin Command Set]]** used by the host to **configure controller or namespace behavior** at runtime.

---
#  1) Purpose

It allows software (OS/driver/tools) to modify parameters such as:

- Power management
- Queue behavior
- Error handling
- Write caching
- Interrupt configuration
- APST (Autonomous Power State Transitions)

👉 Think:

> “Set Features = write configuration knobs into the SSD”

#  2) Command Identification

- **Opcode**: `0x09` (Admin command set)

#  3) Command Structure (SQE Fields)

Key fields inside the Submission Queue Entry:

```
DW10:
  bits 7:0   → Feature Identifier (FID)
  bits 31:8  → Reserved

DW11:
  Feature-specific value (new setting)

DW12–DW15:
  Optional / feature-specific parameters
```

Optional:

- **PRP/SGL** → used if feature requires data buffer

#  4) Completion Result

On success, the controller may return:

```
DW0 (CQE):
  Result field → feature-specific response value
```

#  5) Common Feature Identifiers (FID)

Here are the most important ones:

|FID|Feature|
|---|---|
|0x01|Arbitration|
|0x02|Power Management|
|0x03|LBA Range Type|
|0x04|Temperature Threshold|
|0x05|Error Recovery|
|0x06|Volatile Write Cache|
|0x07|Number of Queues|
|0x08|Interrupt Coalescing|
|0x09|Interrupt Vector Configuration|
|0x0A|Write Atomicity|
|0x0B|Asynchronous Event Configuration|
|0x0C|APST (Autonomous Power State Transitions)|
|0x0D|Host Memory Buffer|
|0x0E|Timestamp|
|0x10+|Vendor-specific / extended|

#  6) Example: Set Power State

To change power state:

```
FID = 0x02 (Power Management)
DW11 = Power State Value (PS index)
```

Example:

```
DW11 = 0x00000003 → set PS3
```

#  7) Example: Enable APST

```
FID = 0x0C (APST)

DW11 = 1 → enable APST
Data buffer (PRP) → APST table
```

#  8) Example: Enable Write Cache

```
FID = 0x06 (Volatile Write Cache)

DW11 bit 0:
  1 → enable
  0 → disable
```


#  9) Persistence Behavior

Features can be:

- **Volatile** → reset after power cycle
- **Non-volatile** → persist across resets

Controlled via:

- Feature definition
- Save bit (in some features)

#  10) Relationship with Get Features

- **Set Features** → configure value
- **Get Features (Opcode 0x0A)** → read current value


#  11) Error Handling

Common failures:

- Invalid FID
- Unsupported feature
- Invalid parameter value

Returned via:

- NVMe status codes in CQE


#  12) Real Usage (Linux Example)

Using nvme-cli:

```
nvme set-feature /dev/nvme0 -f 2 -v 3
# -f feature-id
# -f value
```

👉 Sets power state to PS3

#  13) Key Insight

- Set Features modifies **controller behavior without firmware changes**
- It’s essential for:
    - performance tuning
    - power optimization
    - system integration

#  One-Line Summary

**NVMe Set Features is an admin command (opcode 0x09) that allows the host to configure SSD behavior by writing feature-specific parameters identified by a Feature ID (FID).**

---
## Submission Queue EXAMPLE for Set Feature for APST command

#  Scenario

- Command: **Set Features**
- Feature: **APST (FID = 0x0C)**
- Enable APST
- Provide APST table via PRP


SQE Layout (16 DWORDs)
```
DWORD 0 : OPC | FUSE | CID
DWORD 1 : NSID

DWORD 2 : Reserved
DWORD 3 : Reserved

DWORD 4 : MPTR (low)
DWORD 5 : MPTR (high)

DWORD 6 : PRP1 (low)
DWORD 7 : PRP1 (high)

DWORD 8 : PRP2 (low)
DWORD 9 : PRP2 (high)

DWORD10 : FID
DWORD11 : Feature Value (APST enable)
DWORD12 : Reserved / Feature-specific
DWORD13 : Reserved
DWORD14 : Reserved
DWORD15 : Reserved
```

#  Example Values (Hex Dump)

Assume:

- CID = `0x1234`
- NSID = `0` (controller-level feature)
- PRP1 = `0x0000001000000000` (APST table buffer)
- APST enabled

SQE DWORD-by-DWORD
```
DW0  : 0x00091234
DW1  : 0x00000000

DW2  : 0x00000000
DW3  : 0x00000000

DW4  : 0x00000000
DW5  : 0x00000000

DW6  : 0x00000000   (PRP1 low)
DW7  : 0x00000010   (PRP1 high)

DW8  : 0x00000000
DW9  : 0x00000000

DW10 : 0x0000000C   (FID = 0x0C → APST)
DW11 : 0x00000001   (Enable APST)

DW12 : 0x00000000
DW13 : 0x00000000
DW14 : 0x00000000
DW15 : 0x00000000
```

#  Field Breakdown

## ✔ DW0 (Command Header)

```
0x00091234
```

- OPC (opcode) = `0x09` → Set Features
- CID (command ID) = `0x1234`


## ✔ DW1

```
NSID = 0
```

- APST is **controller-scoped**, so NSID = 0

## ✔ DW6–DW7 (PRP1)

```
PRP1 = 0x00000010_00000000
```

- Points to APST table in host memory


## ✔ DW10 (Feature ID)

```
FID = 0x0C
```

- APST feature

## ✔ DW11 (Feature Value)

```
0x00000001
```

- Bit 0 = 1 → APST enabled

#  APST Table (Referenced by PRP1)

At the memory pointed by PRP1:

```
Entry 0:
  IdleTime = 50 ms
  PowerState = PS1

Entry 1:
  IdleTime = 500 ms
  PowerState = PS2
```

Encoded as NVMe-defined structure.


#  Completion Queue Entry (CQE)

On success:

```
DW0 : Result (optional)
DW3 : Status = SUCCESS
```

#  Key Observations

- **Opcode (0x09)** defines Set Features
- **FID (DW10)** selects APST
- **DW11** enables/disables feature
- **[[Physical Region Page (PRP)]]1** is required to pass APST policy table

#  One-Line Summary

A Set Features APST SQE uses **OPC=0x09, FID=0x0C in DW10, enable flag in DW11, and PRP1 pointing to the APST table**, with all fields packed into a 16-DWORD submission queue entry.


