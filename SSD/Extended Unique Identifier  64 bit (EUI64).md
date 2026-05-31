

## 🧠 Definition

**EUI-64 (Extended Unique Identifier – 64-bit)** is a standardized identifier format defined by the **IEEE**, used to uniquely identify hardware objects. **64-bit globally unique identifier assigned to a namespace**

- Size: **8 bytes (64 bits)**
- Stored in: **Identify Namespace data structure**
- Field: `EUI64`

## 📦 Where It Appears in NVMe

Inside the **Identify Namespace ([[Controller or Namespace Structure (CNS)]] = 0x00)** response:
```
Byte Offset: 120–127
Field:       EUI64
Length:      8 bytes
```

This field provides a **globally unique ID for the namespace**.

## 🔍 Structure of EUI-64

EUI-64 is derived from the IEEE **OUI (Organizationally Unique Identifier)**:
```
| 24-bit OUI (Vendor ID) | 40-bit Extension (vendor assigned) |
```

#### Example (hex):
```
AC-DE-48-00-11-22-33-44
```

- `AC-DE-48` → Vendor OUI
- Remaining → Vendor-defined unique portion


## 🎯 Purpose in NVMe

EUI-64 is used for:

### 1. 🌍 Global Namespace Identification

- Uniquely identifies a namespace across:
    - Systems
    - Controllers
    - Reboots

### 2. 🔄 Persistent Identification

- Unlike **NSID**, which is controller-local, EUI-64 is **globally stable**

### 3. 🧩 Multipath / Multi-controller environments

- Helps OS/storage stack recognize:
    - Same namespace exposed via different paths


## ⚖️ EUI-64 vs NGUID vs NSID

|Identifier|Size|Scope|Persistence|Use Case|
|---|---|---|---|---|
|**NSID**|32-bit|Controller-local|May change|Queue commands|
|**EUI-64**|64-bit|Global|Stable|Cross-system identity|
|**NGUID**|128-bit|Global|Stable|More modern, larger ID|

👉 Important:

- A namespace may have:
    - Only EUI-64
    - Only NGUID
    - Both
- If both exist, **NGUID is generally preferred**


## 🚫 When EUI-64 May Be Zero

If the device doesn’t support or assign it:
```
EUI64 = 00 00 00 00 00 00 00 00
```

In that case:

- Use **NGUID** instead (if available)


## 🔧 Practical Example

Linux (`nvme-cli`):
```
nvme id-ns /dev/nvme0n1
```

#### Output snippet:
```
eui64 : 0025385a01438abc
nguid : 00000000000000000000000000000000
```

## 🧩 Key Takeaway

> **EUI-64 is a globally unique, persistent 64-bit identifier for an NVMe namespace, derived from IEEE vendor space, used for stable identification across systems.**

---

If you want, I can break down:

- Exact **bit-level layout of Identify Namespace (DW fields)**
- How OS uses EUI-64 in **multipath (ANA / NVMe-oF)**
- How EUI-64 compares to **UUIDs in filesystems**