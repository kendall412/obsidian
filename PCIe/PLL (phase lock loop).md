
> **PLL (Phase-Locked Loop)** is one of the most critical circuits in a PCIe PHY. It is an **analog/mixed-signal circuit** that generates the high-speed clocks needed for PCIe transmission and recovers timing from incoming data.

Without a PLL, a PCIe device cannot:

- Transmit serial data at multi-gigabit speeds
- Recover the clock from received data
- Complete LTSSM link training
- Reach the L0 state

## Why PCIe Needs a PLL

The platform typically supplies only a **100 MHz reference clock (REFCLK\#)**.

```
Host

100 MHz REFCLK
        |
        |
        v
     PCIe Device
```

However, a PCIe Gen4 x1 lane transmits at:

```
16 GT/s
```

A 100 MHz clock is far too slow to directly transmit at that rate. The PCIe PHY must generate a much higher-frequency internal clock. This is the PLL's job.

## Where the PLL Is Located

Inside an NVMe SSD:

```
+--------------------------------+
|      NVMe Controller           |
|                                |
| +---------------------------+  |
| | PCIe PHY                  |  |
| |                           |  |
| |  PLL                      |  |
| |  Serializer               |  |
| |  Deserializer             |  |
| |  CDR                      |  |
| +---------------------------+  |
+--------------------------------+
```

The PLL is part of the **PCIe PHY**, not the NVMe controller logic.

## Basic Function

The PLL takes:

```
100 MHz
```

and generates a stable, much higher-frequency clock.

Conceptually:

```
100 MHz

↓

PLL

↓

8 GHz

↓

Serializer
```

The exact internal frequencies depend on the PCIe generation and PHY architecture, but the key idea is that the PLL multiplies and stabilizes the reference timing for high-speed operation.

## Why It's Called "Phase-Locked"

A PLL continuously compares:

- The incoming reference clock
- Its own generated clock

It adjusts itself until both are synchronized in frequency and phase.

Conceptually:

```
Reference Clock

↓

Phase Detector

↓

Control Loop

↓

Oscillator

↓

Generated Clock
```

This feedback loop keeps the generated clock locked to the reference.

## Simplified PLL Block Diagram

```
        REFCLK
        100 MHz
           |
           v
+-------------------+
| Phase Detector    |
+-------------------+
           |
           v
+-------------------+
| Loop Filter       |
+-------------------+
           |
           v
+-------------------+
| Voltage-Controlled|
| Oscillator (VCO)  |
+-------------------+
           |
           v
 High-Speed Clock
```

The PLL continually adjusts the oscillator to stay synchronized.

## PLL During Transmission

Host supplies:

```
100 MHz REFCLK
```

PLL generates:

```
High-Speed Internal Clock
```

Serializer:

```
Parallel Data

↓

Serial Bits
```

The serializer uses the PLL-generated clock to transmit bits at the required data rate.

## PLL During Reception

Receiving is different. The transmitter does **not** send a separate clock wire.

Instead, the receiver gets only:

```
Serial Data
```

Example:

```
101101001001101...
```

The receiver must determine:

- Where each bit begins
- When to sample the data

The PLL works together with the **[[CDR (Clock Data Recovery)]]** circuit.

```
Incoming Bits

↓

CDR

↓

PLL

↓

Recovered Clock

↓

Deserializer
```

## PLL and CDR

These terms are often confused.

### PLL

Uses:

```
Reference Clock
```

to generate the transmit clock.

### CDR

Uses:

```
Incoming Data Stream
```

to recover the receive clock. Most PCIe PHYs integrate these functions closely.

## During LTSSM

After:

```
PERST# Released
```

the LTSSM begins:

```
Detect

↓

Polling
```

During Polling:

- TS1 ordered sets are exchanged.
- The receiver locks its CDR.
- The PLL stabilizes.

Only after the PLL is locked can link training continue.

```
REFCLK

↓

PLL Lock

↓

CDR Lock

↓

TS1 Decode

↓

TS2

↓

Configuration

↓

L0
```

## What Does "PLL Lock" Mean?

A locked PLL means:

```
Generated Clock

=

Reference Timing
```

The frequency and phase are stable enough for reliable communication.

Without lock:

```
Clock Drifts

↓

Incorrect Sampling

↓

Bit Errors
```

The PCIe link cannot operate correctly.

## Relation to [[ASPM (Active State Power Management)]]

When the link enters:

```
L1.2
```

the platform may stop the reference clock.

```
REFCLK

OFF
```

The PLL loses its input and powers down.

Later:

```
CLKREQ#

↓

REFCLK Restored

↓

PLL Relocks

↓

L0
```

This is why wake-up latency includes the time needed for the PLL to lock again.

## PLL Failure Example

Suppose the PLL cannot lock.

```
PERST# Released

↓

Polling

↓

PLL Never Locks

↓

No Symbol Recovery

↓

No TS1 Decode

↓

LTSSM Stuck

↓

Link Training Fails
```

Symptoms include:

- Link never reaches L0
- Repeated Recovery attempts
- PCIe device not detected during enumeration

## Real NVMe Startup

```
Power Applied

↓

REFCLK Stable

↓

PERST# Released

↓

PLL Starts

↓

PLL Locks

↓

Serializer Active

↓

Deserializer Active

↓

TS1

↓

TS2

↓

Configuration

↓

L0
```

## PLL vs Oscillator

|Oscillator|PLL|
|---|---|
|Generates a fixed clock|Generates a clock locked to a reference|
|Free-running|Feedback controlled|
|Lower precision|Very accurate frequency/phase relationship|
|Cannot easily multiply frequency|Can multiply the reference clock|

# Summary

A **PLL (Phase-Locked Loop)** is a **high-speed clock-generation and synchronization circuit** inside the PCIe PHY.

Its roles are:

1. **Transmit:** Multiply the 100 MHz reference clock into the high-speed clocks needed by the serializer.
2. **Receive:** Work with the CDR to recover timing from the incoming serial data stream.
3. **Link Training:** Achieve a stable clock before the LTSSM can progress to L0.

A simplified startup sequence is:

```
100 MHz REFCLK
        |
        v
      PLL
        |
 High-Speed Clock
        |
 Serializer / Deserializer
        |
 LTSSM Link Training
        |
        v
       L0
```

For PCIe and NVMe firmware engineers, messages such as **"PLL lock failed"**, **"PLL relock timeout"**, or **"PLL unlock during Recovery"** usually point to problems in the PHY, reference clock, signal integrity, or power-management transitions rather than in the NVMe controller itself.


---

## PLL relations wrt LTSSM

The **PLL (Phase-Locked Loop)** is **not part of the LTSSM**.

Instead:

- **PLL** is a hardware block inside the **PCIe PHY (Physical Layer)**.
- **LTSSM (Link Training and Status State Machine)** is a digital state machine that controls PCIe link initialization and maintenance.

The LTSSM **depends on** the PLL, but the PLL is **not part of** the LTSSM.

## Relationship

```
             PCIe Controller

+--------------------------------------+
|                                      |
|  LTSSM                               |
|                                      |
|  Detect                              |
|  Polling                             |
|  Configuration                       |
|  Recovery                            |
|  L0                                  |
|                                      |
+--------------------------------------+
                |
                | controls
                v
+--------------------------------------+
|             PCIe PHY                 |
|                                      |
|  PLL                                |
|  CDR                                |
|  Serializer                         |
|  Deserializer                       |
|  Equalizer                          |
|                                      |
+--------------------------------------+
```

Notice:

- LTSSM is above the PHY.
- PLL is inside the PHY.

## Think of the LTSSM as the "Manager"

The LTSSM makes decisions like:

```
Start Detect

↓

Start Polling

↓

Transmit TS1

↓

Transmit TS2

↓

Enter L0
```

But it **cannot** perform these actions directly. It tells the PHY what to do.

## Think of the PLL as the "Engine"

The PLL simply generates the clocks.

```
100 MHz REFCLK

↓

PLL

↓

High-Speed Clock
```

It doesn't know anything about:

- Detect
- Polling
- Recovery
- Configuration

It only provides stable timing.

## Startup Sequence

Let's follow what happens after power-up.

### Step 1: Power Applied

```
Power

↓

REFCLK Stable
```

The PLL receives:

```
100 MHz
```

### Step 2: PERST# Released

```
PERST# ↑
```

Now the LTSSM starts.

```
LTSSM

↓

Detect
```

### Step 3: LTSSM Requests PHY Activity

The LTSSM instructs the PHY:

```
Start Transmitter

↓

Start Receiver
```

The PHY begins enabling:

- PLL
- Serializer
- Receiver
- CDR

### Step 4: PLL Locks

The PHY reports:

```
PLL Locked
```

Only now can reliable high-speed transmission begin.

### Step 5: LTSSM Continues

```
Detect

↓

Polling

↓

TS1

↓

TS2

↓

Configuration

↓

L0
```

Notice: The LTSSM **waits** until the PHY is ready.

## Another Analogy

Imagine a car.

### LTSSM

Driver.

```
Driver

↓

Go Forward
```

### PLL

Engine.

```
Engine

↓

Generate Power
```

The driver isn't the engine.

But without the engine:

```
No Movement
```

Similarly:

Without the PLL:

```
No High-Speed Clock

↓

No TS1

↓

No Link Training
```

## Does the LTSSM Control the PLL?

Yes, indirectly.

For example:

```
Enter L1.2
```

LTSSM tells the PHY:

```
Power Down PHY
```

The PHY may:

```
Stop PLL
```

Later:

```
Wake Link
```

LTSSM requests:

```
Enable PHY
```

PHY:

```
Restart PLL

↓

PLL Lock

↓

Ready
```

### Example During Recovery

Suppose signal quality degrades. LTSSM detects errors.

```
L0

↓

Recovery
```

Recovery may require:

- Re-equalization
- Receiver retraining

The PHY may need to:

- Adjust CDR
- Reconfigure equalization
- Maintain PLL lock (or briefly relock, depending on the implementation)

Again:

The LTSSM controls the process. The PHY performs the electrical work.

## What Happens if the PLL Doesn't Lock?

Suppose:

```
PERST#

↓

Detect
```

PHY:

```
PLL

↓

No Lock
```

The LTSSM cannot continue because the PHY cannot reliably transmit or receive TS1 ordered sets.

The result might be:

```
Detect

↓

Polling

↓

Timeout

↓

Retry

↓

Link Failure
```

The exact recovery behavior depends on the implementation and where the failure occurs, but the link will not successfully reach L0 without a functioning PLL.

## Division of Responsibilities

### LTSSM

Responsible for:

- Detect
- Polling
- Configuration
- Recovery
- L0
- L1
- L2
- Retraining

### PHY

Responsible for:

- PLL
- CDR
- Serializer
- Deserializer
- Equalization
- Electrical signaling

## Communication Between Them

```
LTSSM

↓

Enable PHY

↓

PHY

↓

PLL Lock

↓

PHY Ready

↓

LTSSM

↓

Transmit TS1
```

The LTSSM waits for status signals from the PHY.

## Firmware Engineer's Perspective

If you see a debug log like:

```
PLL Lock Timeout
```

that usually indicates a **PHY-level** problem.

If you see:

```
LTSSM Stuck in Polling
```

the root cause **could** still be a PLL issue, because without a locked PLL the PHY cannot correctly exchange TS1/TS2 ordered sets.

# Summary

The **PLL is not part of the LTSSM**.

Instead:

- **PLL** is an **analog/mixed-signal hardware block** inside the **PCIe PHY** that generates and synchronizes the clocks needed for high-speed serial communication.
- **LTSSM** is a **digital state machine** that manages PCIe link initialization, training, power management, and recovery.

Their relationship can be summarized as:

```
LTSSM   |   | Controls   vPCIe PHY   |   +--> PLL   +--> CDR   +--> Serializer   +--> Deserializer
```

The LTSSM **depends on** the PHY being operational. A locked PLL is one of the prerequisites that allows the LTSSM to successfully progress through Detect, Polling, Configuration, and ultimately reach the **L0** operational state.
