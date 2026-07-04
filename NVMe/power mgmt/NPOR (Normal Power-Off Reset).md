
In NVMe, **NPOR** commonly refers to:

#  Normal Power-On Reset

> It means the controller experienced a **full power cycle reset**, where power was removed and then restored under normal operating conditions. **NPOR (Normal Power-On Reset) is a full power-cycle reset event where the NVMe SSD loses volatile state, reboots firmware, reconstructs FTL state, and restores persistent storage operation.**

#  What Happens During NPOR

During an NPOR event:

```
Power removed
   ↓
Controller state lost
   ↓
Power restored
   ↓
Controller reinitializes
```

The SSD performs:

- Controller reset
- Firmware boot
- DRAM initialization
- FTL recovery
- Metadata reconstruction
- Queue reinitialization

#  What Is Lost During NPOR

Anything volatile:

|Component|Preserved?|
|---|---|
|DRAM cache|❌|
|Queue state|❌|
|In-flight commands|❌|
|NAND data|✅|
|Persistent metadata|✅|

#  NPOR vs Other Reset Types

|Reset Type|Description|
|---|---|
|NPOR|Full power cycle|
|CC.EN reset|Controller disable/enable|
|PCIe FLR|Function-level reset|
|NSSR|NVM subsystem reset|
|Warm reset|No full power loss|
#  Why NPOR Matters

NPOR is important because the SSD must recover safely from:

- Unexpected shutdowns
- System reboot
- AC power cycle
- Datacenter failover

#  NPOR Recovery Process

Typical sequence:

```
1. Boot ROM starts
2. Firmware loads
3. NAND scan
4. FTL journal replay
5. Mapping table reconstruction
6. Bad block table restore
7. Namespace restore
8. Controller becomes READY
```

#  Relation to Unsafe Shutdowns

If power loss was unexpected:

```
Unsafe Shutdown
   ↓
Next boot = NPOR recovery path
```

Controller may:

- replay journals
- recover metadata
- validate mapping consistency

#  NPOR in Validation / Testing

SSD validation teams test:

- NPOR recovery correctness
- Power-loss resilience
- Atomicity after NPOR
- FTL consistency after reset

Common test:

```
Write workload
→ sudden power removal
→ NPOR
→ verify no corruption
```

#  Related SMART Metrics

NPOR-related indicators:

- Power Cycles
- Unsafe Shutdowns
- Media Errors

#  Important Clarification

NPOR is **industry terminology**, not a primary NVMe command or register field.

You’ll commonly see it in:

- firmware logs
- validation docs
- SSD bring-up/debugging
- RTL/firmware discussions

---
# How to test NPOR

> A typical NVMe NPOR test repeatedly power-cycles the SSD, reinitializes the controller, and verifies that firmware, namespaces, FTL mappings, and stored data recover correctly after a full reboot.

A **typical NPOR (Normal Power-On Reset) test** in NVMe validates that the SSD can correctly recover from a **full power cycle** while maintaining:

- controller functionality
- namespace integrity
- FTL consistency
- proper initialization behavior

Unlike SPOR testing, NPOR assumes:

```
Power loss is orderly or expected
```

but the SSD still undergoes a complete reboot and state reconstruction.

#  Purpose of NPOR Testing

Verify that after power cycling:

```
SSD boots correctly
Queues initialize correctly
Namespaces enumerate correctly
Data remains accessible
No firmware/FTL corruption occurs
```

#  What NPOR Tests Validate

|Area|Validation|
|---|---|
|PCIe enumeration|Device reappears correctly|
|Controller init|CSTS.RDY becomes ready|
|Namespace recovery|NSIDs intact|
|FTL reconstruction|Mapping valid|
|SMART counters|Correct persistence|
|Firmware state|Proper boot|
|Queue handling|No stale queue state|
|Data integrity|Previously written data readable|

#  Typical NPOR Test Flow

# 1) Precondition Drive

Apply workload:

```
Sequential writes
Random writes
Mixed R/W
Trim operations
Queue depth stress
```

# 2) Record Baseline State

Capture:

```
SMART log
Error log
Firmware slot info
Namespace info
Known data patterns
```

# 3) Graceful Shutdown or Power Cycle

Typical NPOR event:

```
Power removed cleanly
Wait several seconds
Power restored
```

Unlike SPOR:

- no abrupt removal during NAND program required

# 4) PCIe Re-enumeration

Host verifies:

```
PCIe link trains
NVMe controller detected
BAR accessible
```

# 5) Controller Initialization

Host waits for:

```
CSTS.RDY = 1
```

after setting:

```
CC.EN = 1
```

# 6) Recreate Admin/I/O Queues

Driver performs:

```
Admin queue init
Identify controller
Create IO queues
```

# 7) Namespace Verification

Host checks:

```
Identify Namespace
Namespace count
Capacity
LBA format
```

# 8) Data Integrity Validation

Read previously written patterns:

```
Pattern AAAAAAAA
Pattern walking-1
Random seed data
```

Verify:

```
No corruption
No missing blocks
```

# 9) FTL Recovery Validation

Ensure:

- no mapping corruption
- no metadata inconsistency
- no media errors introduced

# 10) Repeat Cycling

Typical validation runs:

```
100
1,000
10,000+
power cycles
```

to validate long-term reliability.

#  Example NPOR Validation Script

```
loop:
  write random data
  flush
  power cycle
  reinitialize controller
  verify data
  check SMART/errors
```

#  Common Failure Modes Found

## ❌ Controller fails to become ready

```
CSTS.RDY timeout
```

## ❌ Namespace missing

```
Identify Namespace failure
```

## ❌ FTL corruption

```
Read returns wrong data
```

## ❌ Firmware boot failure

```
Controller stuck during init
```

## ❌ SMART counter corruption

Persistent counters incorrect after reboot.

#  NPOR vs SPOR Testing

|Aspect|NPOR|SPOR|
|---|---|---|
|Power loss|Graceful/full cycle|Sudden/unexpected|
|Timing sensitivity|Lower|Very high|
|PLP dependence|Lower|Critical|
|Atomicity stress|Moderate|Severe|
|FTL recovery complexity|Moderate|High|

#  Enterprise Qualification

Enterprise SSD vendors heavily stress:

- repeated NPOR cycles
- thermal NPOR
- NPOR during workload
- NPOR during firmware activation

---
#  Key Insight

> NPOR testing validates the SSD’s ability to completely reboot and reconstruct persistent state correctly after losing all volatile controller state.