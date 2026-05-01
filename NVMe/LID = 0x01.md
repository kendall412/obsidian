
In NVMe, the **Error Information Log Page (LID = 0x01)** is the primary mechanism to retrieve **records of failed or problematic commands**.

---

## 🧠 Definition

> **LID = 0x01 returns a circular log of error entries maintained by the controller.**

- Accessed via **Get Log Page (Admin opcode 0x02)**
- Scope: **controller-wide**
- Each entry corresponds to a **command that completed with an error status**

---

## 📦 Structure Overview

- Log is composed of **N entries** (controller-defined, e.g., 64 or 256)
- Each entry is **64 bytes**
- Entries are ordered by **Error Count (monotonic counter)**

## 🔍 Error Log Entry Layout (64 bytes)

Below is the canonical NVMe structure (simplified but precise):
```
Byte  0–7   : Error Count (64-bit)
Byte  8–9   : Submission Queue ID (SQID)
Byte 10–11  : Command ID (CID)
Byte 12–13  : Status Field (SF)
Byte 14–15  : Parameter Error Location
Byte 16–23  : LBA (if applicable)
Byte 24–27  : Namespace ID (NSID)
Byte 28     : Vendor Specific Info Available
Byte 29–31  : Reserved
Byte 32–63  : Command Specific Information / Reserved
```

## 🧩 Field-Level Meaning

### 🔹 Error Count

- 64-bit counter
- Increments for each error
- Helps detect **new vs old entries**

---

### 🔹 SQID (Submission Queue ID)

- Which queue the command came from
- Useful in multi-queue NVMe systems

---

### 🔹 CID (Command Identifier)

- Matches the original command
- Correlates with **SQE → CQE**

---
### 🔹 Status Field (SF)

- Same format as **Completion Queue Entry status**
- Contains:
    - **Status Code (SC)**
    - **Status Code Type (SCT)**

Examples:

- `0x0000` → Success (usually not logged)
- `0x4004` → Invalid Field
- `0x4001` → Invalid Opcode

---
### 🔹 Parameter Error Location

- Indicates which **DWORD/byte/bit** caused the error
- Critical for debugging malformed commands

---

### 🔹 LBA

- Logical Block Address related to error
- Valid for read/write/media errors

---

### 🔹 NSID (Namespace ID)

- Identifies affected namespace

---

### 🔹 Vendor Specific

- If set, additional info may be available via vendor logs
---
## 🔄 Behavior

- Acts like a **circular buffer**
- Old entries are overwritten when full
- Host must periodically read to avoid losing history

---
## 🔧 Practical Retrieval

Using **nvme-cli**:
`nvme error-log /dev/nvme0`

Example output:
```
Error Log Entries for device:nvme0 entries:64
Entry[ 0]
  error_count     : 15
  sqid            : 1
  cmdid           : 123
  status_field    : 0x4004 (Invalid Field)
  parm_err_loc    : 0x0002
  lba             : 0
  nsid            : 1
```

---
## ⚠️ Important Notes

- **Successful commands are NOT logged**
- Some controllers may log **only critical errors**
- Log depth is **limited → must be polled regularly**

---

## ⚖️ Related Logs

- **SMART / Health (LID 0x02)** → aggregate error counters
- **Telemetry logs (0x07/0x08)** → deep debugging
- **Vendor logs** → extended error context

---

## 🧩 Key Insight

> **The Error Information Log is your first line of visibility into NVMe command failures—mapping directly back to the offending command, queue, and address.**


If you want, I can:

- Decode **Status Field bit-by-bit (SCT/SC/DNR/M bits)**
- Map an error log entry back to a **specific SQE (DW0–DW15)**
- Show real-world debugging scenarios (e.g., invalid PRP vs media error)