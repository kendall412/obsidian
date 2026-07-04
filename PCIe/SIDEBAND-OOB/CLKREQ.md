
> **CLKREQ# (Clock Request)** is one of **PCIe [[sideband signals]]** that allows a PCIe device to **request the PCIe reference clock (REFCLK)** from the platform. It is primarily used for **power management**.

The `#` means the signal is **[[active low signal]]**:

- **CLKREQ# = Low (asserted)** → "I need the reference clock."
- **CLKREQ# = High (deasserted)** → "I don't currently need the reference clock."

## Why Does PCIe Need CLKREQ#?

A PCIe device (such as an NVMe SSD) requires a **100 MHz reference clock (REFCLK)** to operate its PHY.

```
REFCLK (100 MHz)
        |
        v
+--------------------+
| PCIe PHY           |
| PLL                |
| Serializer          |
| Deserializer        |
+--------------------+
```

Keeping the reference clock running all the time consumes power. In laptops and other low-power systems, the platform can **turn off REFCLK** when the PCIe device is idle. But before the device can communicate again, it needs the clock back. That's what CLKREQ# is for.

## Where CLKREQ# Fits

A simplified PCIe connection:

```
           Root Complex

      +--------------------+
      |                    |
      | REFCLK ----------->|
      |                    |
      |<------ CLKREQ# ----|
      +--------------------+

                |
             PCIe Lanes
                |
      +--------------------+
      | NVMe SSD           |
      +--------------------+
```

- **REFCLK** flows from the platform to the SSD.
- **CLKREQ#** is driven by the SSD to request that the platform provide REFCLK.

## Relationship with [[REFCLK]]

REFCLK:

```
Host
 |
100 MHz Clock
 |
SSD
```

CLKREQ#:

```
SSD
 |
Need Clock?
 |
Host
```

The SSD doesn't generate REFCLK—it requests it.

## Typical Operation

### System Active

```
REFCLK = ON
CLKREQ# = Low
PCIe Link = L0
```

Everything is running normally.

### Idle

Suppose the SSD has been idle.

```
No Reads
No Writes
```

The platform decides:

```
Turn REFCLK Off
```

Power is saved.

### New Read Command Arrives

The SSD needs to become active.

It asserts:

```
CLKREQ# = Low
```

Meaning:

```
Please provide REFCLK.
```

The platform enables:

```
100 MHz REFCLK
```

Once the clock is stable:

```
PHY PLL locks
```

The PCIe link can transition back to:

```
L0
```

## State Diagram

```
             L0

              |
          Idle Time

              v

            L1.2

              |
        REFCLK Removed

              |
              v

Device Needs Service

              |

        CLKREQ# Asserted

              |

        REFCLK Enabled

              |

        PLL Locks

              |

              v

             L0
```

## CLKREQ# and ASPM

CLKREQ# is commonly used together with **[[ASPM (Active State Power Management)]]**.

Example:

```
L0
 |
No Traffic
 |
v
L1.2
 |
Clock Removed
 |
CLKREQ#
 |
Clock Restored
 |
L0
```

Without CLKREQ#, the platform would not know when to restore the clock.

### Example: Laptop NVMe SSD

User closes the laptop.

SSD becomes idle.

```
PCIe Link

↓

L1.2
```

Platform:

```
Stops REFCLK
```

Power consumption drops.

Later:

User opens a file.

SSD:

```
Assert CLKREQ#
```

Platform:

```
Enable REFCLK
```

SSD:

```
PLL Locks

↓

L0

↓

Read Command
```

## Timing Sequence

A simplified sequence:

```
Time →

REFCLK

██████████████████____________________██████████

CLKREQ#

____________________\________________/

PCIe Link

L0 ---------> L1.2 ---------> L0
```

Interpretation:

1. Link enters L1.2.
2. Platform removes REFCLK.
3. SSD needs to wake.
4. SSD asserts CLKREQ#.
5. Platform restores REFCLK.
6. Link returns to L0.

## Does CLKREQ# Carry Data?

No. It is a **single digital control signal**.

It does **not** carry:

- PCIe TLPs
- NVMe commands
- User data
- Management data

It only indicates:

```
Need Clock

or

Clock Not Needed
```

## Which Component Drives CLKREQ#?

Typically:

```
NVMe SSD

↓

CLKREQ#

↓

Root Complex / Chipset
```

The device asserts CLKREQ# when it requires the reference clock. The platform monitors the signal and controls the clock.

## Difference Between [[REFCLK]] and CLKREQ#

|REFCLK|CLKREQ#|
|---|---|
|100 MHz clock signal|Request signal|
|Host → Device|Device → Host (typical)|
|Continuous clock waveform|Single control signal|
|Used by the PHY|Used for power management|

## Relationship with Other [[Sideband Signals]]

|Signal|Purpose|
|---|---|
|PERST#|Reset the PCIe device|
|REFCLK|Supply the 100 MHz reference clock|
|CLKREQ#|Request the reference clock|
|PEWAKE#|Wake the platform from sleep|
|SMBus|Out-of-band management interface|

## Important for NVMe Engineers

Many **low-power resume problems** involve CLKREQ#.

Example:

```
Laptop wakes

↓

SSD asserts CLKREQ#

↓

REFCLK never appears

↓

PLL never locks

↓

LTSSM stuck

↓

SSD disappears
```

Firmware and hardware engineers debugging resume failures often probe:

- CLKREQ#
- REFCLK
- PERST#
- LTSSM state

with an oscilloscope or logic analyzer to determine where the wake-up sequence failed.

# Summary

**CLKREQ#** is an **active-low PCIe sideband signal** used for **reference clock power management**.

Its role is:

```
SSD Needs Clock

↓

Assert CLKREQ#

↓

Platform Enables REFCLK

↓

PHY PLL Locks

↓

PCIe Link Returns to L0
```

It works closely with:

- **REFCLK**
- **ASPM (especially L1 substates such as L1.2)**
- **PCIe PHY**

to reduce idle power while allowing the PCIe link to wake up quickly when communication resumes.