> If everything works as expected, the Link trains to the L0 state without ever going into the Recovery state. If the correct Symbol pattern isn’t seen in the Configuration Idle, the LTSSM goes to Recovery in an effort to correct signaling problems by adjusting equalization values. Secondly, once L0 is reached with a data rate of 2.5 GT/s and both devices support higher speeds, the LTSSM goes to Recovery and attempts to change the Link speed to the highest commonly-supported/advertised speed. In this state Bit Lock and either Symbol Lock or Block Alignment is reacquired and the Link is deskewed again. The Link and Lane Numbers should remain unchanged unless the Link width is being changed. In that case, LTSSM passes through the Configuration state where Link width is re-negotiated.

The **Recovery state** is an LTSSM state used when a PCIe link is already established, but the link needs to be **retrained, re-synchronized, or changed** without performing a full reset.

Think of it as:

> "The link is up, but we need to temporarily pause normal traffic and retrain the link."

## Where Recovery Fits in LTSSM

```
Detect
   |
Polling
   |
Configuration
   |
L0  <------------------+
 |                     |
 |                     |
 +-----> Recovery -----+
```

Unlike Detect/Polling/Configuration, which happen during initial link bring-up, **Recovery** usually occurs after the link is already operational.

## Why Does Recovery Exist?

Common reasons:
### 1. Speed Change

The most common reason.

Example:

```
Initial Training:
Gen1 x4
```

After the link reaches L0:

```
L0
 |
 v
Recovery
 |
 v
Gen4 x4
 |
 v
L0
```

The link retrains to a higher speed.

### 2. Equalization Training

For:

- PCIe Gen3
- PCIe Gen4
- PCIe Gen5
- PCIe Gen6

equalization is required.

Recovery is used to perform that training.

### 3. Excessive Errors

Suppose the PHY detects:

```
CRC errors
Symbol errors
Block lock loss
```

The LTSSM may enter:

```
Recovery
```

to restore link quality.

### 4. Software-Initiated Retrain

Software can set the PCIe Retrain Link bit.

Example:

```
Link Control Register
       |
       v
Retrain Link = 1
```

The LTSSM enters Recovery.

## Recovery Substates

The exact substates vary by PCIe generation, but conceptually:

```
Recovery.RcvrLock
        |
        v
Recovery.RcvrCfg
        |
        v
Recovery.Speed
        |
        v
Recovery.Equalization
        |
        v
L0
```

Let's walk through them.

### 1. Recovery.RcvrLock

Goal:

```
Re-establish bit lock
```

The receiver must ensure:

```
CDR Locked
Bit Lock Good
```

Similar to Polling.Active.

Example:

```
Incoming Data
      |
      v
CDR
      |
      v
Clock Recovery
```

### 2. Recovery.RcvrCfg

Goal:

```
Re-establish lane synchronization
```

Verify:

```
Lane Alignment
Symbol Lock
Block Lock
```

All lanes must be synchronized again.

### 3. Recovery.Speed

Used when changing speed.

Example:

Current:

```
Gen1 x4
```

Desired:

```
Gen4 x4
```

The devices exchange training sequences indicating:

```
Move to Gen4
```

The PHY reconfigures:

```
PLL
CDR
Equalization
```

for the new rate.

### 4. Recovery.Equalization

For Gen3 and above.

The transmitter and receiver optimize signal quality.

Example:

```
TX Preset
RX CTLE
RX DFE
```

are adjusted.

Simplified:

```
TX
 |
 +--> Try Preset 0
 +--> Try Preset 1
 +--> Try Preset 2
```

Receiver evaluates which setting works best.

## Example: NVMe SSD Gen4 Link Bring-Up

A common sequence:

### Initial Training

```
Detect
Polling
Configuration
L0
```

Link comes up at:

```
Gen1 x4
```

### Speed Upgrade

```
L0
 |
 v
Recovery
```

Recovery performs:

```
Change Speed
Equalization
```

Result:

```
Gen4 x4
```

### Return to L0

```
Recovery
    |
    v
L0
```

Now the SSD operates at full speed.

# #What Happens During Recovery?

Normal PCIe traffic is temporarily halted.

For example:

Before:

```
Memory Write TLP
Memory Read TLP
Completion TLP
```

During Recovery:

```
No Normal TLP Traffic
```

Instead:

```
Training Sequences
Control Symbols
Equalization Exchanges
```

are transmitted.

Once Recovery completes:

```
Normal Traffic Resumes
```

#@ Recovery vs Polling

They look similar but occur at different times.

|Polling|Recovery|
|---|---|
|Initial link bring-up|Existing link retraining|
|After Detect|After L0|
|Establish first communication|Re-establish communication|
|TS1/TS2 training|TS1/TS2 retraining|
|Link not yet operational|Link was already operational|

## Recovery Failure

If Recovery cannot complete:

Example:

```
Gen4 Equalization Failed
```

Possible outcomes:

### Downtrain

```
Gen4 → Gen3
```

or

```
Gen3 → Gen2
```

### Full Retrain

```
Recovery    |    vDetect
```

The entire link training process restarts.

## What NVMe Engineers Often See

A healthy SSD:

```
Detect
Polling
Configuration
L0
Recovery
L0
```

Meaning:

```
Gen1 training
then
Gen4 retraining
```

This is completely normal.

An unhealthy SSD may repeatedly cycle:

```
L0
 |
 v
Recovery
 |
 v
L0
 |
 v
Recovery
```

This often indicates:

- Signal integrity problems
- Poor equalization
- Bad connector
- Marginal PCB routing
- PHY bugs

## Real Example

Suppose:

```
PCIe Capability

Target Speed = Gen4
```

Initial link:

```
L0 @ Gen1
```

Then:

```
Recovery.Speed
```

changes PHY settings.

Then:

```
Recovery.Equalization
```

optimizes the signal.

Finally:

```
L0 @ Gen4
```

## Summary

The **Recovery state** is a retraining state used after a PCIe link is already up.

Its purposes include:

```
Speed Changes
Equalization
Error Recovery
Retraining
Lane Re-synchronization
```

Typical flow:

```
L0
 |
 v
Recovery.RcvrLock
 |
 v
Recovery.RcvrCfg
 |
 v
Recovery.Speed
 |
 v
Recovery.Equalization
 |
 v
L0
```

For a typical NVMe SSD, Recovery is commonly used immediately after the first L0 entry to move from the initial **Gen1 x4** link to the final operating speed such as **Gen4 x4** or **Gen5 x4**.