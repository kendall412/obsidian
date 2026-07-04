
> **Sideband signals** are **dedicated electrical signals that exist outside the high-speed PCIe lanes**. They are used for **control, reset, power management, clock management, wake-up, and system management** rather than for transferring PCIe packets (TLPs).

Think of a PCIe connection as two groups of signals:

```
High-Speed Data Signals
    PCIe TX/RX Differential Pairs
            +
Sideband Signals
    Reset
    Clock
    Wake
    SMBus
```

The PCIe lanes carry TLPs, while the sideband signals coordinate and manage the link.

## PCIe Connector Overview

A simplified PCIe connection looks like:

```
                 CPU / Root Complex
              +----------------------+
              |                      |
              | PCIe Controller      |
              +----------------------+
                 |             |
     PCIe Lanes  |             | Sideband Signals
                 |             |
                 v             v

          =========================
          || TX/RX Lanes         ||
          =========================
                 |
          +----------------------+
          |   NVMe SSD           |
          +----------------------+
```

The PCIe lanes transfer data. The sideband signals manage the connection.

## Common PCIe Sideband Signals

The most common sideband signals are:

|Signal|Direction (Typical)|Purpose|
|---|---|---|
|PERST#|Host → Device|Reset PCIe device|
|REFCLK+ / REFCLK−|Host → Device|Reference clock|
|CLKREQ#|Device ↔ Host|Request reference clock|
|PEWAKE#|Device → Platform|Wake the system|
|SMBus (SMCLK/SMDAT)|Bidirectional|Management interface|
|PRSNT#|Slot ↔ Motherboard|Card presence detection (add-in cards)|

Let's examine each one.

## 1. [[PERST]]# (PCIe Reset)

**PERST#** stands for **PCI Express Reset**. It is one of the most important sideband signals.

```
Host
 |
PERST#
 |
SSD
```

Purpose:

- Holds device in reset
- Initializes PCIe controller
- Starts PCIe initialization

Example startup:

```
Power Applied
      |
PERST# asserted
      |
Power stabilizes
      |
PERST# deasserted
      |
LTSSM starts
      |
Detect
Polling
Configuration
L0
```


## 2. [[REFCLK]]\#

PCIe devices require a stable reference clock.

```
Host
 |
100 MHz REFCLK
 |
SSD
```

The reference clock is used by the PHY to generate the high-speed serial bit clock.

Typical frequency:

```
100 MHz
```

## 3. [[CLKREQ]]\#

**CLKREQ#** means:

```
Clock Request
```

Purpose:

Allow the device to request that the platform supply the PCIe reference clock.

Example:

Laptop idle:

```
Reference Clock OFF
```

SSD needs service:

```
CLKREQ# asserted

↓

Clock enabled

↓

PCIe wakes
```

This helps reduce idle power.

## 4. [[PEWAKE]]\#

PEWAKE# means:

```
PCIe Wake
```

Purpose:

Allow the PCIe device to wake the platform.

Example:

```
SSD
 |
PEWAKE#
 |
Chipset
 |
CPU wakes
```

Used for:

- Wake from sleep
- Power management events

## 5. SMBus

SMBus is also considered a sideband interface.

```
BMC
 |
SMBus
 |
NVMe SSD
```

Used for:

- Temperature
- SMART
- Inventory
- Firmware information
- NVMe-MI

Unlike [[PERST]]#, SMBus carries actual management messages.

## 6. [[PRSNT]]\#

Mostly used on PCIe add-in cards.

Purpose:

Detect whether a PCIe card is inserted.

```
Slot

Card inserted?

↓

PRSNT# asserted
```

Desktop systems commonly use this.

## Sideband vs PCIe Lanes

PCIe lanes carry:

```
TLP
DLLP
Ordered Sets
```

Sideband signals carry:

```
Reset
Clock
Wake
Management
```

No TLPs travel over PERST# or CLKREQ#.

### Example: Power-On Sequence

```
Power Applied
      |
REFCLK stable
      |
PERST# asserted
      |
PERST# released
      |
LTSSM Detect
      |
Polling
      |
Configuration
      |
L0
```

Notice: The PCIe lanes are not active until after PERST# is released.

### Example: Idle Power Saving

System idle:

```
PCIe Link

↓

L1.2
```

Reference clock removed:

```
CLKREQ#

↓

Clock Off
```

New NVMe Read arrives:

```
SSD

↓

CLKREQ#

↓

Clock Restored

↓

L0
```

### Example: NVMe Management

Normal data:

```
CPU

↓

PCIe

↓

SSD
```

Management:

```
BMC

↓

SMBus

↓

NVMe-MI

↓

SSD
```

SMBus is the sideband interface carrying the management traffic.

## Which Signals Carry Data?

|Signal|Carries PCIe Packets?|
|---|---|
|PCIe TX/RX Lanes|Yes|
|PERST#|No|
|REFCLK|No|
|CLKREQ#|No|
|PEWAKE#|No|
|SMBus|Management data only|
## Enterprise Server Example

```
               CPU
                |
             PCIe x4
                |
             NVMe SSD
                ^
                |
             SMBus
                |
               BMC

PERST# --------->

REFCLK --------->

CLKREQ# <--------

PEWAKE# -------->
```

Everything except the PCIe lanes is considered a sideband connection.

## Sideband Signals During LTSSM

During initialization:

```
Power
 |
REFCLK Stable
 |
PERST#
 |
LTSSM Starts
 |
Detect
Polling
Configuration
L0
```

During power management:

```
L0

↓

L1.2

↓

CLKREQ#

↓

Clock Removed
```

# Summary Table

|Sideband Signal|Purpose|Carries Data?|
|---|---|---|
|PERST#|Reset PCIe device|No|
|REFCLK|Provide reference clock|No|
|CLKREQ#|Request reference clock|No|
|PEWAKE#|Wake system|No|
|SMBus|Management communication|Yes (management only)|
|PRSNT#|Detect card presence|No|

# Key Takeaway

The **PCIe lanes** are responsible for transporting **Transaction Layer Packets (TLPs)**, which carry memory reads, memory writes, completions, and other PCIe traffic.

**Sideband signals** are separate electrical connections that support the PCIe ecosystem by handling:

- **Initialization** (PERST#)
- **Clocking** (REFCLK, CLKREQ#)
- **Power management** (CLKREQ#, PEWAKE#)
- **Device management** (SMBus/NVMe-MI)
- **Presence detection** (PRSNT#)

> For an NVMe firmware engineer, the most important sideband signals are **PERST#**, **REFCLK**, **CLKREQ#**, and **SMBus**, because they directly affect device initialization, low-power operation, and management.

