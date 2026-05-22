
> PCIe link training is the process where two PCIe components establish a working communication link after reset or power-up.

The two sides are:

- **Upstream component**  
    Usually:
    - CPU root complex,
    - PCIe switch upstream port.
- **Downstream component**  
    Usually:
    - NVMe SSD,
    - GPU,
    - NIC,
    - FPGA card.

The process is controlled by:

Link Training and Status State Machine (LTSSM).

# Goal of PCIe Link Training

The link must negotiate:

|Parameter|Examples|
|---|---|
|Lane count|x1, x4, x8, x16|
|Link speed|Gen1–Gen6|
|Clock synchronization|Bit timing|
|Equalization|Signal integrity tuning|
|Polarity inversion|Lane correction|
|Lane numbering|Lane mapping|
|Encoding synchronization|Symbol alignment|

Only after training completes can:

- PCIe packets (TLPs),
- MMIO,
- DMA,
- configuration transactions,  
    occur.

# High-Level PCIe Link Training Flow

```
Power On / Reset
        |
        v
Detect
        |
        v
Polling
        |
        v
Configuration
        |
        v
Equalization (Gen3+)
        |
        v
L0 (Link Up)
```

All of this is orchestrated by the LTSSM.

# Step-by-Step PCIe Link Training

# 1. Detect State

Both devices start in:

```
LTSSM = Detect
```

Purpose:

- determine whether another PCIe device exists.

## Electrical Detection

PCIe transmitter sends detection signals.

Receiver termination is checked.

If proper impedance detected:

```
Device detected
```

# 2. Polling State

Once both sides detect each other:

```
LTSSM -> Polling
```

Purpose:

- synchronize serial communication.

## Transmitting [[Ordered Sets]]

Both sides exchange special training sequences:

```
TS1
TS2
```

These are not normal packets.

They contain:

- lane info,
- link number,
- speed capability,
- polarity info,
- equalization parameters.

# TS1 and TS2

Training Sequences are repetitive symbols.

Purpose:

- clock recovery,
- lane alignment,
- parameter negotiation.

# 3. Bit Lock / Clock Recovery

PCIe is high-speed serial communication.

Receiver must recover:

- embedded clock,
- bit boundaries.

This is called:

```
CDR (Clock Data Recovery)
```

Without successful CDR:

- link cannot continue.

# 4. Lane Polarity Detection

PCIe allows differential pair inversion.

Example:

```
TX+ <-> TX-
```

Receiver automatically detects/corrects this during training.

# 5. Lane Numbering / Lane Reversal

For multi-lane links:

```
x4
x8
x16
```

the physical lane order may be reversed on PCB routing.

PCIe can automatically reorder lanes.

Example:

```
Physical:
Lane 3 2 1 0

Logical:
Lane 0 1 2 3
```


# 6. Configuration State

Now devices negotiate:

|Capability|Example|
|---|---|
|Maximum speed|Gen4|
|Lane width|x4|
|Encoding mode|8b/10b, 128b/130b|
|Equalization support|Gen3+|

Final common denominator selected.

Example:

```
Root complex: Gen5 x16
SSD:          Gen4 x4

Final link:
Gen4 x4
```

# 7. Equalization (Gen3 and Above)

High-speed PCIe links suffer:

- attenuation,
- reflections,
- ISI (Inter-Symbol Interference),
- crosstalk.

Equalization compensates.

# PCIe Equalization

Introduced heavily in:

- Gen3,
- Gen4,
- Gen5,
- Gen6.

Uses:

- transmitter de-emphasis,
- receiver CTLE,
- DFE tuning.


# Equalization Phases

Gen3+ typically uses:

|Phase|Purpose|
|---|---|
|Phase 0|Preset exchange|
|Phase 1|Receiver evaluation|
|Phase 2|Coefficient adjustment|
|Phase 3|Finalization|

Devices tune signal quality dynamically.


# 8. Link Width Negotiation

If some lanes fail:

```
x4 device
```

may fall back to:

```
x2 or x1
```

Example:

- damaged lane,
- bad connector,
- SI issue.

# 9. Speed Negotiation

PCIe usually starts at lowest speed:

|Generation|Raw Rate|
|---|---|
|Gen1|2.5 GT/s|

Then attempts speed upgrades progressively:

```
Gen1 -> Gen2 -> Gen3 -> Gen4
```

If higher speed unstable:

- link falls back.

# 10. Enter L0 State

When training succeeds:

```
LTSSM -> L0
```

This means:

```
LINK UP
```

Normal PCIe traffic now allowed:

- TLPs,
- DLLPs,
- MMIO,
- DMA,
- MSI-X.

# Recovery State

If errors occur later:

```
LTSSM -> Recovery
```

Link may:

- retrain,
- reduce speed,
- re-equalize.

# PCIe Generations and Training Complexity

| Generation | Complexity             |
| ---------- | ---------------------- |
| Gen1/2     | Relatively simple      |
| Gen3       | Equalization added     |
| Gen4       | More aggressive tuning |
| Gen5       | Extremely SI-sensitive |
| Gen6       | PAM4 + FEC complexity  |

# Real NVMe Example

An Non-Volatile Memory Express SSD:

```
PCIe Gen4 x4
```

Power-up sequence:

```
CPU Root Port
    |
Detect SSD
    |
Exchange TS1/TS2
    |
Train at Gen1
    |
Speed up to Gen4
    |
Perform equalization
    |
Enter L0
    |
NVMe enumeration begins
```

Only after L0 can:

- BARs be read,
- config space accessed,
- NVMe driver initialize queues.

# Important Training Signals

| Signal/Concept            | Purpose                     |
| ------------------------- | --------------------------- |
| TS1                       | Training sequence           |
| TS2                       | Training sequence           |
| EIOS                      | Electrical idle ordered set |
| SKP                       | Clock compensation          |
| COM symbol                | Alignment                   |
| Equalization coefficients | SI tuning                   |
# What Happens If Training Fails

Possible outcomes:

| Failure              | Result               |
| -------------------- | -------------------- |
| No receiver detected | No link              |
| Clock recovery fails | Polling loop         |
| Equalization fails   | Speed downgrade      |
| Too many bad lanes   | Width downgrade      |
| Severe SI problems   | Link never enters L0 |

# Debugging PCIe Link Training

Common debug tools:

| Tool              | Purpose               |
| ----------------- | --------------------- |
| LTSSM trace       | State transitions     |
| Protocol analyzer | TS1/TS2 inspection    |
| Oscilloscope      | Signal integrity      |
| BERT              | Bit error testing     |
| PCIe analyzer     | Equalization analysis |
# Simplified Mental Model

PCIe link training is essentially:

```
Two high-speed serial devices
learning how to communicate reliably
before real traffic begins
```


# Key Takeaway

PCIe link training is the automated negotiation and synchronization process that establishes a functional PCIe link.

It includes:

- device detection,
- clock recovery,
- lane alignment,
- speed negotiation,
- equalization,
- width negotiation,
- error recovery.

The LTSSM manages the entire process until the link reaches:

```
L0 = fully operational
```

where normal PCIe communication can begin.

