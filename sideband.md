
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

