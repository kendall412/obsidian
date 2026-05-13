
It is a **128-bit (16-byte) identifier** used to uniquely identify a **namespace across all systems, controllers, and time**.

## 🧠 Where NGUID Appears

NGUID is part of the **Identify Namespace data structure** returned by the Identify command:
```
Byte Offset: 104–119
Field:       NGUID
Length:      16 bytes (128 bits)
```

## 🔍 What NGUID Represents

- A **globally unique, persistent identifier**
- Assigned by the **vendor**
- Remains **constant** regardless of:
    - Controller attachment
    - PCIe path changes
    - Reboots


## 🧩 Structure

Unlike EUI-64, NGUID does **not strictly follow IEEE OUI format**.  
It is typically:
`Vendor-defined 128-bit unique value`

#### Example:
`60 1C 3A 92 7F 4B 11 EE 9A 3C 02 42 AC 12 00 02`

Some vendors may internally base it on:

- UUID/GUID schemes
- Extended EUI formats
- Random or sequential allocation (with uniqueness guarantees)

---
## 🎯 Purpose

### 1. 🌍 Global Namespace Identity

- Uniquely identifies a namespace **across the entire infrastructure**

### 2. 🔄 Multipath & NVMe-oF

- Critical in:
    - NVMe over Fabrics (NVMe-oF)
    - Multi-controller subsystems
- Helps OS determine:
    > “These multiple paths map to the same underlying namespace”
    

### 3. 🧷 Persistent Binding

- Used by:
    - Operating systems
    - Storage orchestration layers
- For stable device mapping (e.g., `/dev/disk/by-id` in Linux)

## ⚖️ NGUID vs EUI-64 vs NSID
|Identifier|Size|Scope|Stability|Notes|
|---|---|---|---|---|
|**NSID**|32-bit|Controller-local|Not guaranteed|Used in commands|
|**EUI-64**|64-bit|Global|Stable|IEEE-based|
|**NGUID**|128-bit|Global|Stable|Preferred modern ID|

👉 **Key rule:**

- If **NGUID ≠ 0**, it is typically **preferred over EUI-64**
- If both are zero → no global ID provided

---
## 🚫 When NGUID Is Zero

If unsupported:
`NGUID = 00...00 (16 bytes)`

Then:

- Use **EUI-64** if available
- Otherwise fall back to **NSID (less reliable globally)**


## 🔧 Practical Example (Linux)

Using `nvme-cli`:
```
nvme id-ns /dev/nvme0n1
```

output:
```
nguid : 601c3a927f4b11ee9a3c0242ac120002
eui64 : 0025385a01438abc
```



## 🧠 Key Insight

> **NGUID is the most robust, globally unique identifier for an NVMe namespace and is the preferred method for persistent identification in modern systems.**

---

If you want, I can go deeper into:

- Bit-level layout of the **Identify Namespace structure (DW-by-DW)**
- How NGUID is used in **NVMe multipath (ANA states)**
- Differences between **NGUID and UUID List (CNS = 0x15)**
