#evt #dvt #fvt #pvt
In SSD development, **EVT, DVT, FVT, and PVT** are validation phases used to move a product from early engineering samples to mass production.

The exact meaning can vary by company, but commonly:

| Stage | Meaning | Main Purpose |
|---|---|---|
| **EVT** | Engineering Validation Test | Prove the basic engineering design works |
| **DVT** | Design Validation Test | Prove the design meets product/spec requirements |
| **FVT** | Firmware Validation Test / Functional Validation Test | Validate firmware features, functionality, and corner cases |
| **PVT** | Production Validation Test | Prove the factory/manufacturing process is ready |

Typical flow:

```text
EVT → DVT → FVT → PVT → MP
```

Where **MP** means **Mass Production**.

---

## 1. EVT — Engineering Validation Test
#evt
**EVT** is the early bring-up and engineering validation phase.

The goal is to check whether the SSD hardware, controller, NAND, firmware, and basic system architecture work.

### In SSD development, EVT includes:

- Controller silicon or FPGA bring-up
- PCIe link training
- NVMe enumeration
- Firmware boot
- NAND detection and initialization
- DRAM initialization, if used
- Basic FTL operation
- Basic read/write testing
- Power-on reset validation
- Basic admin and I/O command testing
- Initial thermal and power checks
- Early SPOR/NPOR behavior checks

### EVT answers:

```text
Can the SSD basically function?
Can the host detect it?
Can it read and write data?
Are there major hardware or firmware blockers?
```

At EVT, many bugs are expected. Debug firmware, special test hooks, reworked boards, and engineering samples are common.


## 2. DVT — Design Validation Test
#dvt
**DVT** validates that the SSD design meets the product requirements.

This phase is more formal and much broader than EVT.

### In SSD development, DVT includes:

- NVMe specification compliance
- PCIe compliance
- Signal integrity testing
- Performance validation
- Power state validation
- Thermal throttling validation
- Endurance testing
- Data integrity testing
- Error handling
- Bad block management
- NAND retry/read-disturb handling
- Power-loss recovery testing
- SPOR and NPOR validation
- Secure erase, sanitize, format testing
- Firmware update testing
- Compatibility with different hosts, BIOS, OS, and platforms
- Temperature and voltage corner testing

### DVT answers:

```text
Does the SSD design meet the specification?
Is it reliable across real-world conditions?
Are the controller, NAND, PCB, and firmware design acceptable?
```


## 3. FVT — Firmware Validation Test / Functional Validation Test
#fvt
**FVT** usually focuses on validating the SSD firmware and product functions.

Different companies use the term differently:

- **Firmware Validation Test**
- **Functional Validation Test**
- Sometimes part of DVT
- Sometimes a separate phase between DVT and PVT

### In SSD development, FVT includes:

- NVMe command behavior
- FTL correctness
- Garbage collection behavior
- Wear leveling
- Bad block management
- Read retry
- ECC handling
- RAID/parity recovery, if used
- Thermal throttling firmware logic
- SMART log behavior
- Error log behavior
- Firmware download and activation
- Namespace management
- Power management
- Sanitize and secure erase
- SPOR/NPOR recovery logic
- Background scan
- Data retention handling
- QoS and latency behavior
- Corner-case command sequences

### FVT answers:

```text
Does the firmware behave correctly?
Do all features work as intended?
Are corner cases handled safely?
```

Example FVT test cases:

```text
Write data → SPOR → power on → verify data integrity
Run GC during heavy I/O → reset controller → verify no data loss
Trigger thermal limit → verify throttling works
Run firmware update → reboot → verify new firmware active
```



## 4. PVT — Production Validation Test
#pvt
**PVT** validates that the product can be built consistently in the factory.

The design should already be mostly stable before PVT. The focus moves from engineering design to manufacturing readiness.

### In SSD development, PVT includes:

- Factory production flow validation
- Test time optimization
- NAND sorting/binning validation
- Controller and NAND programming flow
- Firmware loading process
- Serial number and label programming
- Manufacturing test coverage
- Yield analysis
- Burn-in or stress screening
- Final QA checks
- Packaging validation
- Lot-to-lot consistency
- Golden sample comparison
- Capacity configuration validation
- Production firmware validation

### PVT answers:

```text
Can we manufacture this SSD repeatedly with good yield?
Is the factory test process stable?
Is the product ready for mass production?
```



## Simple comparison

| Phase | Focus | SSD Example |
|---|---|---|
| **EVT** | Basic engineering bring-up | SSD enumerates as NVMe and can read/write |
| **DVT** | Design/spec validation | SSD passes NVMe, PCIe, power, thermal, endurance tests |
| **FVT** | Firmware/function validation | FTL, GC, wear leveling, SPOR recovery, SMART logs work correctly |
| **PVT** | Production readiness | Factory can build, test, program, and ship SSDs reliably |



## More practical SSD lifecycle

```text
Architecture / Planning
        ↓
Controller bring-up / NAND selection
        ↓
EVT
        ↓
DVT
        ↓
FVT
        ↓
PVT
        ↓
MP
```

Sometimes companies use:

```text
EVT → DVT → PVT → MP
```

and include **FVT** inside **DVT** or across all stages.



## Short summary

- **EVT**: Does the SSD basically work?
- **DVT**: Does the SSD design meet the spec?
- **FVT**: Does the SSD firmware/functionality work correctly?
- **PVT**: Can the SSD be manufactured reliably?
- **MP**: Mass production release.