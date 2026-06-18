
> In NVMe and PCIe, **sideband signals** are signals that are **separate from the main PCIe data lanes**.

The PCIe lanes carry:

```
TLPs
DLLPs
NVMe Commands
Read Data
Write Data
```

Sideband signals carry control, reset, clock, power-management, and presence information.

## Main Data Path vs Sideband

For an M.2 NVMe SSD:

```
                PCIe Link
        +-----------------------+
Host <--| Lane0 Lane1 Lane2 Lane3 |--> SSD
        +-----------------------+

            Main Data Path
```

Separate from that:

```
PERST#
CLKREQ#
PEWAKE#
REFCLK
```

These are sideband signals.

## Why Are Sideband Signals Needed?

Some functions must occur **before** the PCIe link exists.

Example:

```
Reset SSD
```

You cannot send a PCIe packet to reset the SSD because:

```
PCIe link not trained yet
```

Instead:

```
Host drives PERST#
```

which is a sideband signal.

## Common NVMe Sideband Signals

For M.2 NVMe SSDs, the most important sideband signals are:

|Signal|Purpose|
|---|---|
|PERST#|PCIe reset|
|REFCLK+/-|Reference clock|
|CLKREQ#|Clock request / power management|
|PEWAKE#|Wake host from low-power state|
|Presence Detect (platform dependent)|Device presence|

### 1. [[PERST]] (PCIe Reset)

Most important sideband signal.

```
PERST# = 0
```

means:

```
SSD held in reset
```

When released:

```
PERST# = 1
```

the SSD begins:

```
LTSSM Detect
Polling
Configuration
L0
```

No PCIe traffic is needed.

### 3. CLKREQ\#

Used for power management.

Example:

```
System enters low-power mode
```

Reference clock may be disabled.

Later SSD needs clock:

```
CLKREQ# asserted
```

Host re-enables clock.

```
SSD
 |
 +--> CLKREQ#
 |
Host
 |
 +--> REFCLK ON
```

### 4. PEWAKE\#

Power Event Wake.

Allows SSD to wake the platform.

Example:

```
System sleeping
```

SSD detects an event.

SSD asserts:

```
PEWAKE#
```

Host wakes up.

Common in modern power-management architectures.

# Sideband vs In-Band

#### Sideband

Outside PCIe packets.

Examples:

```
PERST#
REFCLK
CLKREQ#
PEWAKE#
```

Physical pins.

#### In-Band

Inside PCIe protocol.

Examples:

```
TLP
DLLP
MSI-X
Hot Reset
PM Messages
```

Travel over PCIe lanes.

# Example During Boot

Power-on sequence:

```
Power Rails Stable
        |
        v
REFCLK Active
        |
        v
PERST# Released
        |
        v
LTSSM Detect
        |
        v
Polling
        |
        v
Configuration
        |
        v
L0
```

Notice:

```
REFCLK
PERST#
```

are sideband signals.

No PCIe packets exist yet.

# Example During Normal Operation

Once link is up:

```
Host
 |
Memory Write TLP
 |
PCIe Lanes
 |
SSD
```

This is **not** sideband.

It's normal PCIe traffic.

# Example: NVMe Controller Reset

There are two ways:

### Sideband Reset

```
PERST# asserted
```

Resets:

```
PHY
PCIe Controller
LTSSM
NVMe Controller
```

### In-Band Reset

```
CC.EN = 0
```

written through PCIe MMIO.

Requires:

```
Link already in L0
```

This is not sideband.

# M.2 NVMe Example

Typical important signals:

```
M.2 Connector

PCIe TX/RX Lanes
PCIe TX/RX Lanes
PCIe TX/RX Lanes
PCIe TX/RX Lanes

REFCLK+
REFCLK-

PERST#
CLKREQ#
PEWAKE#
```

Everything except the lanes is generally considered sideband.

# Why Firmware Engineers Care

Many SSD startup problems involve sideband signals:

### PERST# Issue

```
LTSSM never starts
```

### REFCLK Issue

```
CDR (Bit Lock) cannot lock
```

### CLKREQ# Issue

```
Power-state failures
```

### PEWAKE# Issue

```
Wake-from-sleep failures
```

The PCIe link may look broken even though the root cause is actually a sideband signal.

# Summary

A **sideband signal** in NVMe/PCIe is a control signal that exists outside the PCIe data lanes and protocol.

Common NVMe sideband signals:

|Signal|Function|
|---|---|
|**PERST#**|Hardware reset|
|**REFCLK**|100 MHz reference clock|
|**CLKREQ#**|Request clock for power management|
|**PEWAKE#**|Wake host from low-power state|

These signals are essential because they enable:

```
Reset
Clocking
Power Management
Wake Events
```

even before the PCIe link is trained and before any NVMe commands can be exchanged.