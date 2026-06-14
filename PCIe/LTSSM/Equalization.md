
> **Equalization** is the process of adjusting the transmitter and receiver signal characteristics so that high-speed PCIe data can be received reliably. For **PCIe Gen3 and higher**, equalization is a critical part of the **Recovery** state because the signal has degraded while traveling through:

- PCB traces
- Connectors
- M.2 sockets
- Backplanes
- Retimers

Without equalization, Gen3/Gen4/Gen5 links often cannot operate reliably.

## Why Equalization Is Needed

Imagine a transmitted signal:

```
Transmitter_|‾|_|‾|_|‾|_
```

After traveling through a motherboard trace:

```
Receiver_/‾\__/‾\__/‾\_
```

The edges become rounded and smaller.

At Gen4:

```
16 GT/s
```

each bit period is only:

which is about **62.5 ps** per bit.

A distorted signal can easily cause bit errors.

## What Equalization Does

Equalization tries to "undo" channel loss.

```
Transmitter
     |
     v
Signal Conditioning
     |
     v
PCB Channel
     |
     v
Receiver Conditioning
     |
     v
Clean Eye Diagram
```

## Where Equalization Happens

There are two sides:

### Transmitter Equalization

The transmitter intentionally shapes the signal.

```
TX
 |
 +--> Pre-cursor
 +--> Main Cursor
 +--> Post-cursor
```

## Receiver Equalization

The receiver compensates for loss.

Examples:

```
CTLE
DFE
```

## What Happens During LTSSM Recovery?

Suppose the link initially trains:

```
Detect
Polling
Configuration
L0 @ Gen1
```

Now both devices want Gen4.

The LTSSM enters:

```
Recovery
```

and performs equalization.

## Recovery.Equalization Process

Simplified:

```
L0 @ Gen1
      |
      v
Recovery.Speed
      |
      v
Switch to Gen4 Signaling
      |
      v
Recovery.Equalization
```

Now the two devices exchange training information.

## Transmitter Presets

PCIe defines multiple transmit presets.

Example:

```
Preset 0
Preset 1
Preset 2
...
Preset 10
```

Each preset changes:

```
Pre-cursor
Main Cursor
Post-cursor
```

strengths.

Example:

```
Preset A
```

may emphasize high-frequency content more than:

```
Preset B
```

## Receiver Evaluation

The receiver measures:

```
Eye Height
Eye Width
BER
Signal Quality
```

Example:

```
Try Preset 3
      |
      v
Eye Too Small

Try Preset 6
      |
      v
Eye Good
```

The receiver tells the transmitter which setting works best.

## Eye Diagram Concept

Before equalization:

```
Eye Opening

 \  /
  \/
  /\
 /  \
```

Small eye.

Hard to distinguish 1s and 0s.

After equalization:

```
Eye Opening

\      /
 \    /
  \  /
  /  \
 /    \
/      \
```

Large eye.

Reliable communication.

## Gen3/Gen4 Example

For an NVMe SSD:

```
CPU
 |
Motherboard
 |
M.2 Connector
 |
NVMe SSD
```

The PCIe channel may lose significant high-frequency energy.

Without equalization:

```
Gen4 BER too high
```

With equalization:

```
Gen4 BER acceptable
```

and the link can remain at Gen4.

## If Equalization Fails

Suppose the receiver tests every supported preset.

```
Preset 0 -> Fail
Preset 1 -> Fail
Preset 2 -> Fail
...
Preset N -> Fail
```

The LTSSM may decide:

```
Gen4 Not Possible
```

and downtrain:

```
Gen4 -> Gen3
```

or even:

```
Gen3 -> Gen2
```

## Real LTSSM Flow

For a Gen4 NVMe SSD:

```
PERST#
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
L0 @ Gen1
   |
   v
Recovery.Speed
   |
   v
Recovery.Equalization
   |
   +--> Exchange Training Information
   +--> Evaluate Presets
   +--> Tune TX/RX
   |
   v
L0 @ Gen4
```

## What Firmware Engineers See

Many SSD controllers expose debug information such as:

```
LTSSM State
EQ Phase
TX Preset
RX CTLE
BER
```

Typical log:

```
Enter Recovery.Equalization
Preset 5 Selected
CTLE Gain 3
Equalization PASS
Enter L0 Gen4
```

or:

```
Enter Recovery.Equalization
Equalization FAIL
Fallback Gen3
```

## Equalization vs Link Training

People often say:

```
"PCIe link training"
```

This includes:

```
Receiver Detection
TS1/TS2 Exchange
Lane Numbering
Speed Negotiation
Equalization
```

Equalization is one specific part of the overall training process, primarily used for high-speed links.

## NVMe Example

Suppose:

```
Gen4 NVMe SSD
Gen4 Root Complex
```

Expected:

```
16 GT/s x4
```

The link initially comes up:

```
Gen1 x4
```

Then Recovery performs equalization.

If successful:

```
L0 @ Gen4 x4
```

If unsuccessful:

```
L0 @ Gen3 x4
```

This is one of the most common reasons a Gen4-capable NVMe SSD operates at Gen3 speed.

## Summary

**Equalization during the LTSSM Recovery stage** is the process where the PCIe transmitter and receiver tune their signal-conditioning parameters so that high-speed communication (Gen3/Gen4/Gen5) is reliable.

```
L0 @ Gen1
      |
      v
Recovery.Speed
      |
      v
Recovery.Equalization
      |
      +--> Choose TX Presets
      +--> Adjust RX CTLE/DFE
      +--> Measure Signal Quality
      |
      v
L0 @ Target Speed
```

If equalization succeeds, the link stays at the higher generation. If it fails, the LTSSM typically downtrains to a lower speed that can be operated reliably.