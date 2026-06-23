
> A PCIe Gen4 device may train or retrain to a lower generation (Gen3, Gen2, or even Gen1) when the link cannot reliably operate at Gen4 speeds.

For a Gen4 NVMe SSD, the most common outcome is:

```
Expected: Gen4 x4 (16 GT/s per lane)
Actual:   Gen3 x4 (8 GT/s per lane)
```

This process is called **downtraining**.

### 1. Signal Integrity Problems (Most Common Cause)

Gen4 operates at:

```
16 GT/s per lane
```

which is much more demanding than Gen3.
If the receiver cannot reliably recover data, Gen4 training may fail.

Common causes:

#### Excessive Trace Loss

```
Root Complex
      |
 Long PCB Trace
      |
 NVMe SSD
```

Signal becomes too weak at Gen4 frequencies.

#### Poor PCB Layout

Examples:

- Impedance discontinuities
- Via stubs
- Poor return paths
- Improper differential pair routing

#### Connector Problems

Examples:

- Worn M.2 connector
- Poor insertion
- Oxidation
- Manufacturing defects

### 2. [[Equalization]] Failure

For Gen3 and above, PCIe requires equalization.

During:

```
L0 @ Gen1
      |
      v
Recovery.Equalization
```

the devices try to find transmit and receive settings that work.

Examples:

```
TX Preset
TX De-emphasis
RX CTLE
RX DFE
```

If no acceptable settings are found:

```
Gen4 Equalization FAIL
```

the link may fall back to:

```
Gen3
```

instead.

This is extremely common in PCIe debugging.

### 3. Clock Problems

PCIe Gen4 requires tighter clock quality.

Problems include:

- Excessive jitter
- Noisy reference clock
- SSC issues
- PLL instability

Example:

```
100 MHz Refclk
       |
       v
Excessive Jitter
       |
       v
CDR Cannot Lock Reliably
```

Result:

```
Gen4 training failure
```

### 4. Lane Quality Differences

Suppose a x4 NVMe SSD:

```
Lane0 Good
Lane1 Good
Lane2 Good
Lane3 Marginal
```

At Gen3:

```
All lanes work
```

At Gen4:

```
Lane3 BER too high
```

The LTSSM may:

```
Retry
Retry
Downtrain
```

to Gen3.

### 5. Firmware or PHY Configuration Bugs

Examples:

#### Incorrect Equalization Settings

```
TX Preset Wrong
```

#### Wrong PHY Initialization

```
CTLE Gain Incorrect
```

#### LTSSM Bug

```
Recovery.Equalization never completes
```

Result:

```
Fallback to Gen3
```

## 6. Motherboard BIOS Limitations

Some systems intentionally limit speed.

Examples:

```
BIOS forces Gen3
```

or:

```
PCIe Compatibility Mode
```

Then:

```
Gen4 SSD
+
Gen4 Root Complex

=
Gen3 Link
```

even though the hardware supports Gen4.

## 7. Retimers and Redrivers

Servers often contain:

```
CPU
 |
Retimer
 |
Backplane
 |
SSD
```

If the retimer has issues:

- Firmware mismatch
- Equalization problems
- Signal degradation

the link may negotiate lower speeds.

## 8. Thermal Problems

Less common, but possible.

Examples:

```
PHY overheating
PLL unstable
```

Some platforms may intentionally retrain to a lower speed.

## 9. Electrical Noise

Examples:

```
Power supply noise
Ground bounce
EMI
```

At Gen4, the eye opening is much smaller than Gen3. A noisy system may pass Gen3 but fail Gen4.

## 10. Excessive Bit Error Rate (BER)

Suppose the link initially reaches:

```
Gen4 x4
```

but experiences many errors:

```
Replay events
CRC failures
Block lock loss
```

LTSSM may enter:

```
Recovery
```

and eventually settle at:

```
Gen3 x4
```

for stability.

## Real NVMe Example

Normal sequence:

```
PERST# released
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
```

If equalization succeeds:

```
L0 @ Gen4 x4
```

If equalization fails:

```
L0 @ Gen3 x4
```

## How to Check

On Linux:

```
lspci -vv
```

Example:

```
LnkCap: Speed 16GT/s, Width x4
LnkSta: Speed 8GT/s, Width x4
```

Interpretation:

```
Supported = Gen4
Current   = Gen3
```

The hardware can do Gen4, but the link trained at Gen3.

## NVMe Debugging Clues

If you see:

```
Supported: Gen4 x4
Current:   Gen3 x4
```

investigate:

1. Signal integrity
2. Equalization logs
3. LTSSM recovery events
4. Reference clock quality
5. Connector quality
6. PCB routing
7. Retimer status
8. BIOS speed settings
# Most Common Cause

For Gen4 NVMe SSDs in real systems, the most common root causes are:

```
1. Signal integrity problems
2. Equalization failure
3. Poor connector/backplane quality
4. Retimer issues
```

These account for the majority of cases where a Gen4-capable device falls back to Gen3 during PCIe link training.

