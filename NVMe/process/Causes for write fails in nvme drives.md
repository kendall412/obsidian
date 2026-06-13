
> An NVMe SSD failing to write isn’t a single failure mode—it’s usually one of several layers (host, protocol, firmware, or media) blocking writes. You want to isolate whether the drive is **intentionally refusing writes** (protective behavior) or **unable to write** (hardware/path issue).

# 1. Drive-level protection states (most common)

These are _by design_—the SSD is protecting itself or its data.
### Read-only / write-protect mode

- Triggered by:
    - NAND wear-out (endurance exhausted)
    - Internal media errors exceeding thresholds
- Behavior:
    - Reads still work
    - Writes fail or return errors

**How to detect**

- NVMe SMART log:
    - `critical_warning` bit (especially bit 0: available spare below threshold)
    - `percentage_used` near or above 100%

### Thermal throttling → write blocking

- If temperature exceeds limits, firmware may:
    - Throttle heavily
    - Temporarily block writes

**Check**

- SMART temperature log
- Composite temperature vs threshold

### Namespace write protection

- Namespace may be explicitly write-protected
- Causes:
    - Firmware config
    - Host command (`Set Features`)
    - Security state

# 2. Firmware / controller issues

### Firmware bug or crash

- Controller firmware may:
    - Hang write queues
    - Stop processing submissions

**Symptoms**

- SQ entries posted, but no CQ completions
- Timeouts on write commands

### FTL (Flash Translation Layer) failure

- Mapping tables corrupted or inconsistent
- Garbage collection stuck
- Journal replay failure

**Effect**

- Writes rejected or never completed

# 3. NAND / media problems

### Bad block exhaustion

- Too many unusable blocks
- No space for remapping

### Program/erase failure

- NAND cannot reliably program cells

**Symptoms**

- Media errors in logs
- Write commands returning failure status

# 4. Capacity / space conditions

### Drive is full (100% utilization)

- No free blocks available
- Especially problematic if:
    - TRIM not working
    - Over-provisioning too low

**Result**

- Writes stall or fail

### No garbage collection headroom

- High write amplification
- GC cannot free space fast enough

# 5. Host / system-side issues

### PCIe link problems

- Link instability → dropped or retried writes

### Driver or OS issues

- NVMe driver bug
- Queue mismanagement

### IOMMU / DMA issues

- Incorrect memory mapping
- Data not reaching device properly

# 6. Command / protocol errors

### Invalid NVMe command

- Wrong:
    - LBA range
    - NLB (number of logical blocks)
    - PRP/SGL mapping

**Result**

- Controller rejects write

### Flush or write ordering issues

- If required flushes aren’t handled:
    - Writes may be delayed or blocked


# 7. Power-related issues

### Power loss protection (PLP) event

- After unsafe shutdown:
    - Drive may enter recovery mode
    - Writes temporarily blocked

### Insufficient power

- Especially in high-performance SSDs
- Voltage droop → controller instability

# 8. Security / administrative locks

### Sanitize / format in progress

- Writes blocked during operation

### Security lock (SED / encryption)

- Drive locked → no writes allowed

---
# How to debug systematically

### Step 1 — Check SMART / health

Look at:

- Critical warnings
- Media errors
- Temperature
- Available spare

### Step 2 — Check NVMe [[status codes (sc)]]

From completion queue:

- `WRITE FAULT`
- `MEDIA ERROR`
- `LBA OUT OF RANGE`

### Step 3 — Verify queue behavior

- Are SQ tail doorbells ringing?
- Are CQ head pointers advancing?


### Step 4 — Check host logs

- Kernel logs (`dmesg`)
- NVMe driver errors

### Step 5 — Test basic writes

- Try small aligned writes
- Try different LBAs

---

# Quick mental model

If an NVMe drive won’t write, it’s usually:

1. **Protecting itself** → wear-out, overheating
2. **Out of usable space** → GC or capacity issue
3. **Controller/firmware stuck**
4. **Host/driver/PCIe problem**

---

If you want, I can walk through a **real NVMe trace (SQE + CQE)** and show exactly how a failed write looks at the register level—that’s typically how engineers debug this in practice.
