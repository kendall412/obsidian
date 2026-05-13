
> In NVMe, **status codes** are how the controller tells the host the result of a command—success, failure, or specific error condition. They are returned in the **Completion Queue Entry (CQE)** for every command.

---

# Where status codes live

A completion entry (CQE) includes a **Status field** (16 bits) that encodes:

- **SCT (Status Code Type)** → category of the error
- **SC (Status Code)** → specific condition
- Additional bits (phase tag, more, do-not-retry)

# Bit-level layout (Status field)

```
15      14      13:12      11:9        8:1         0
+-------+-------+----------+-----------+-----------+
| DNR   | MORE  | Reserved |   SCT     |    SC     |
+-------+-------+----------+-----------+-----------+
```

- **SC (Status Code)** → what went wrong (or success)
- **[[status code type (sct)]]** → which class of errors
- **DNR (Do Not Retry)** → retrying won’t help
- **MORE** → more status info available (rare

# Status Code Types (SCT)

These define _buckets_ of errors:

### 0x0 — Generic Command Status

Most common category

Examples:

- `0x00` → **Success**
- `0x01` → Invalid Command Opcode
- `0x02` → Invalid Field in Command
- `0x04` → Data Transfer Error
- `0x05` → Commands Aborted
- `0x06` → Internal Device Error

### 0x1 — Command-Specific Status

Errors tied to specific commands

Examples:

- Invalid log page
- Invalid firmware slot
- Invalid namespace

### 0x2 — Media and Data Integrity Errors

Actual NAND / storage issues

Examples:

- Uncorrectable read error
- Write fault
- End-to-end (E2E) data protection error

### 0x3 — Path-related Errors

Multi-path / fabric environments (NVMe-oF)

Examples:

- ANA (Asymmetric Namespace Access) issues
- Path unavailable

# Common Status Codes (practical set)

Here are the ones engineers see most:

### Success

- **SCT=0, SC=0x00**
    - Command completed normally


### Invalid command / parameters

- **Invalid Opcode (0x01)** → wrong command
- **Invalid Field (0x02)** → bad parameter
- **LBA Out of Range (0x80)** → invalid address

### Data transfer issues

- **Data Transfer Error (0x04)**
    - PRP/SGL mapping issues
    - DMA failure

### Media errors

- **Write Fault (0x80, SCT=2)**
- **Unrecovered Read Error (0x81, SCT=2)**

### Internal errors

- **Internal Device Error (0x06)**
    - Firmware crash
    - Controller fault

### Aborts / timeouts

- **Command Aborted (0x07)**
- Often due to:
    - Reset
    - Timeout handling

# How to interpret a status (example)

Suppose CQE shows:

- **SCT = 0x2 (Media)**
- **SC = 0x80 (Write Fault)**
- **DNR = 1**

👉 Meaning:

- The write failed due to a media issue
- Retry is pointless (DNR=1)


# Why status codes matter

They’re essential for:

- Debugging failed reads/writes
- Understanding drive health
- Distinguishing:
    - Host bug vs controller bug vs NAND failure

# Mental model

Think of NVMe status like this:

- **SCT = “category of problem”**
- **SC = “exact failure reason”**
- **DNR = “should I retry?”**


If you want, I can decode a **real CQE hex dump** and walk you through exactly how to extract SCT/SC from raw bits—that’s what you’d do in firmware or driver debugging.