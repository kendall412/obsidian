
The **NVMe Get Features command** is the counterpart to _[[Set Features]]_: it lets the host **query the current value and capabilities of a specific feature** from the controller or a namespace.

---

# 🔹 1) Purpose

- Read configuration/state of NVMe features (power, queues, cache, APST, etc.)
- Verify what was set via **[[Set Features]]**
- Discover defaults, limits, and current operating values

👉 Think:

> “Get Features = read configuration knobs from the SSD”

# 🔹 2) Command Identification

- **Opcode**: `0x0A` (Admin command set)

---

# 🔹 3) Command Structure (SQE Fields)

```
DW10:
  bits 7:0   → Feature Identifier (FID)
  bits 31:8  → Select (SEL) + Reserved

DW11:
  Feature-specific (often unused for simple queries)

DW12–DW15:
  Optional / feature-specific

PRP/SGL:
  Used if the feature returns a data buffer
```

## 🔹 Select (SEL) Field (Important)

Located in **DW10[10:8]**, it controls _what value you want_:

|SEL|Meaning|
|---|---|
|0|Current value|
|1|Default value|
|2|Saved value (non-volatile)|
|3|Supported capabilities|

👉 This is a key difference vs Set Features.


# 🔹 4) Completion Result

- Returned in **CQE DW0 (Result field)** for simple features
- Or via **data buffer** (PRP/SGL) for complex features


# 🔹 5) Common Feature Identifiers (FID)

Same as Set Features:

|FID|Feature|
|---|---|
|0x01|Arbitration|
|0x02|Power Management|
|0x06|Volatile Write Cache|
|0x07|Number of Queues|
|0x08|Interrupt Coalescing|
|0x0C|APST|
|0x0D|Host Memory Buffer|
# 🔹 6) Example: Get Power State

```
FID = 0x02 (Power Management)
SEL = 0 (current)
```

Result:

```
CQE DW0 → current power state (PS index)
```

# 🔹 7) Example: Get Write Cache Status

```
FID = 0x06
```

Result:

```
bit 0:
  1 → write cache enabled
  0 → disabled
```

# 🔹 8) Example: Get APST Configuration

```
FID = 0x0C
```

- Returns:
    - APST enabled/disabled (DW0)
    - APST table (via data buffer)

# 🔹 9) Buffer-Based Features

Some features require memory:

```
PRP1 → pointer to host buffer
```

Examples:

- APST table
- Host Memory Buffer (HMB)

# 🔹 10) Relationship with Set Features

```
Set Features → write valueGet Features → read value
```

Typical workflow:

1. Set feature
2. Get feature → verify

# 🔹 11) Real Example (Linux)

Using nvme-cli:

```
nvme get-feature /dev/nvme0 -f 2
```


👉 Returns current power state


# 🔹 12) Key Insight

- Get Features is **non-destructive**
- Used heavily for:
    - debugging
    - validation
    - capability discovery

# 🔹 One-Line Summary

**NVMe Get Features (opcode 0x0A) retrieves the current, default, saved, or supported values of a feature identified by FID, returning results via CQE or a data buffer.**

---

If you want, I can:

- show a **bit-level decode of DW10 (FID + SEL)**
- or walk through a **real nvme-cli output interpretation**