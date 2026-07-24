
> Here is the **bit-accurate layout of an NVMe Completion Queue Entry (CQE)**—this applies to the **Admin Completion Queue (ACQ)** as well (the structure is identical for Admin and I/O CQs; only the _meaning_ of DW0 varies by command).

# NVMe CQE (16 bytes = 4 DWORDs)
```
DW0 : Command-Specific Result
DW1 : Reserved
DW2 : SQ Head Pointer (SQHD) | SQ Identifier (SQID)
DW3 : Command Identifier (CID) | Status Field (SF)
```

# DW0 — Command-Specific Result (32 bits)
```
Bits 31:0 → Command-Specific Result
```

- For **Admin commands**, this often returns meaningful data:
    - e.g., **Identify**, **Get Log Page**, **Set Features**
- For many I/O commands, may be unused or command-dependent

# DW1 — Reserved (32 bits)
```
Bits 31:0 → Reserved (0)
```

- Typically ignored by host

# DW2 — SQ Head Pointer + SQ Identifier
```
Bits 15:0  → SQHD (Submission Queue Head Pointer)
Bits 31:16 → SQID (Submission Queue Identifier)
```

### Meaning

- **[[SQHD (Submission Queue Head Pointer)]]**:
    - Indicates how far the controller has consumed the SQ
    - Helps host reclaim SQ entries
- **SQID**:
    - Identifies which SQ this completion corresponds to

# DW3 — CID + Status Field
```
Bits 15:0  → CID  (Command Identifier)

Bits 31:16 → Status Field (SF)
```

## Status Field (SF) breakdown
```
Bits 31     → P    (Phase Tag)
Bits 30:25  → SC   (Status Code)
Bits 24:17  → SCT  (Status Code Type)
Bits 16     → More (More completions pending)
Bits 15     → DNR  (Do Not Retry)
Bits 14:0   → Reserved
```

⚠️ Note: Bit ordering is sometimes shown differently across docs; the canonical interpretation is:

### Clean logical view:
```
Status Field (16 bits total):

Bit 15 → Phase Tag (P)
Bits 14:9 → Status Code (SC)
Bits 8:6 → Status Code Type (SCT)
Bit 5 → More (M)
Bit 4 → Do Not Retry (DNR)
Bits 3:0 → Reserved
```

## Field meanings

### Phase Tag (P)
- Toggles each time CQ wraps
- Used by host to detect new vs stale entries

### Status Code (SC)

- Indicates result of command:
    - `0x0` → Success
    - others → error codes
### Status Code Type (SCT)
| Value | Meaning              |
| ----- | -------------------- |
| 0     | Generic              |
| 1     | Command-specific     |
| 2     | Media/Data integrity |
| 3     | Path-related         |

# Example CQE (decoded)

```
DW0 : Command-Specific Result
DW1 : Reserved
DW2 : SQ Head Pointer (SQHD) | SQ Identifier (SQID)
DW3 : Command Identifier (CID) | Status Field (SF)
```

```
DW0 = 0x00000001
DW1 = 0x00000000
DW2 = 0x00030010
DW3 = 0xA005002A
```

### Decode:

- **CID** = `0x002A`
- **SQID** = `0x0003`
- **SQHD** = `0x0010`
- **Status Field**:
    - P = 1
    - SC = 0 (success)
    - SCT = 0 (generic)

### Key nuances

#### 1. Same format for Admin and I/O CQs
- Only DW0 interpretation differs

#### 2. CID is critical
- Used to match completion → original SQE

#### 3. Phase Tag drives polling
- Host checks:
    - If phase matches expected → new entry
    - Otherwise → stop