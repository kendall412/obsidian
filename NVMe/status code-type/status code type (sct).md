
> In NVMe, the **Status Code Type (SCT)** classifies **what category of error or completion status** occurred. It is part of the **Status field in the Completion Queue Entry (CQE)**. 
> 
> **SCT in NVMe is a 3-bit field that categorizes completion status into generic, command-specific, media/data, path-related, or vendor-defined error types.** 
> 
> SCT tells you **what category the error belongs to**, while SC tells you **the exact error**.

# 🔹 Where SCT Is Located

In CQE **DW3 (Status field)**:

```
bits 15:9  → Status Code (SC)
bits 8:6   → Status Code Type (SCT)
bits 5:1   → Flags
bit 0      → Phase Tag (P)
```

# 🔹 All SCT Values (per NVMe spec)

## ✔ 0x0 — Generic Command Status

- Applies to **all command sets**
- Most commonly used

Examples:

- Success
- Invalid Opcode
- Invalid Field
- Data Transfer Error

## ✔ 0x1 — Command Specific Status

- Errors specific to a **particular command**

Examples:

- Invalid queue identifier
- Invalid log page
- Invalid firmware slot

## ✔ 0x2 — Media and Data Integrity Errors

- Related to **NAND/media issues or data corruption**

Examples:

- Unrecovered read error
- End-to-end guard check failure
- LBA out of range (in some contexts)

## ✔ 0x3 — Path Related Status

- Used in **multi-path / multi-controller environments**

Examples:

- Asymmetric access state
- Path unavailable
- Controller path errors

## ✔ 0x4 — Reserved

- Not defined in current spec (reserved for future use)

## ✔ 0x5 — Reserved

## ✔ 0x6 — Reserved

## ✔ 0x7 — Vendor Specific

- Used by vendor-defined extensions

👉 Meaning depends on SSD vendor implementation


# 🔹 Summary Table

|SCT|Name|Description|
|---|---|---|
|0x0|Generic|Common command status|
|0x1|Command Specific|Command-related errors|
|0x2|Media/Data Integrity|NAND/data issues|
|0x3|Path Related|Multipath/transport|
|0x4|Reserved|—|
|0x5|Reserved|—|
|0x6|Reserved|—|
|0x7|Vendor Specific|Vendor-defined|

# 🔹 How SCT Is Used

To interpret a completion:

```
Status = (SCT, SC)
```

Example:

```
SCT = 0x0 (Generic)
SC  = 0x00 → SUCCESS
```


---

If you want, I can:

- list **all Status Codes (SC) under each SCT**
- or decode a **real NVMe CQE status field bit-by-bit**
