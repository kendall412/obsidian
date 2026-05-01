

## What PSID Is

The **PSID** is a **unique, factory-programmed identifier** assigned to an NVMe SSD by the manufacturer. It is typically printed **physically on the drive label** and is **not readable via standard NVMe commands** for security reasons.

- Usually a **32-character alphanumeric string**
- Tied to the drive’s **hardware identity**
- Used only for **security recovery operations**

## 🧠 Why PSID Exists

PSID is part of the **self-encrypting drive (SED)** security model, commonly aligned with standards like:

- TCG Opal
- IEEE 1667

These standards allow drives to enforce **encryption and access control** at the hardware level.

---

## 🚨 Primary Use Case: PSID Revert

The **only practical use of PSID** is to perform a:

> **PSID Revert (a.k.a. Secure Factory Reset)**

### When is this needed?

- You forgot the **drive password**
- The drive is **locked/encrypted**
- You cannot access the data via normal authentication

### What does it do?

- **Completely erases all data**
- **Removes encryption keys**
- **Resets the drive to factory state**

⚠️ This operation is **irreversible**.


## 🔧 How It Works (Conceptually)

1. User obtains the **PSID** from the drive label
2. Uses a vendor tool or utilities like:
    - `nvme-cli` (Linux)
    - Manufacturer-specific tools (Samsung Magician, Intel MAS, etc.)
3. Issues a **PSID revert command**
4. Controller verifies PSID → triggers full crypto erase



| Property         | Description                            |
| ---------------- | -------------------------------------- |
| Accessibility    | Not exposed via NVMe Identify or logs  |
| Storage location | Hardcoded in firmware / secure area    |
| Security role    | Bypass authentication for secure erase |
| Data recovery    | Impossible after PSID revert           |

## 🧩 Important Clarification

PSID is **not**:

- A namespace identifier
- A controller ID
- Related to submission/completion queues

It is strictly a **security fallback mechanism**.


---
If you want, I can go deeper into:

- Exact NVMe command flow for PSID revert
- How PSID ties into the **Security Send / Receive commands**
- Bit-level differences between **Format NVM vs Sanitize vs PSID revert**