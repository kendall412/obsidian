
> **PCIe Device Power States** and **NVMe Power States** are completely different power-management mechanisms defined by different specifications.

| PCIe Device Power States       | NVMe Power States                     |
| ------------------------------ | ------------------------------------- |
| Defined by PCIe/PCI PM spec    | Defined by NVMe spec                  |
| Control the PCIe device itself | Control the NVMe controller operation |
| D0, D1, D2, D3hot, D3cold      | PS0, PS1, PS2, PS3, ...               |
| Managed by OS PCI subsystem    | Managed by NVMe driver                |
| Device-wide power state        | Storage-controller power state        |
### Think of an NVMe SSD as Two Layers

```
+-----------------------------+
| NVMe Controller (P states)  |
+-----------------------------+
| PCIe Device (D states)      |
+-----------------------------+
| PCIe Link (L states)        |
+-----------------------------+
```

Power can be managed at each layer independently.

## PCIe Device Power States (D States)

Defined by the PCI Power Management specification.

Purpose:

```
Reduce power of the PCIe device itself
```

States:

```
D0      Fully On
D1      Light Sleep
D2      Deeper Sleep
D3hot   Nearly Off
D3cold  Power Removed
```

### D0

Normal operation.

```
PCIe Controller Active
BAR Registers Accessible
DMA Works
Interrupts Work
```

Typical NVMe operation:

```
PCIe = D0
NVMe = PS0
```

### D3hot

Much of the device is powered down.

```
No normal I/O
Most logic off
Configuration space retained
```

The device is still powered.

### D3cold

Deepest state.

```
Power removed
```

Equivalent to:

```
Device effectively off
```

Returning usually requires:

```
PERST#
Enumeration
Initialization
```

again.

## NVMe Power States (PS States)

Defined by the NVMe specification.

Purpose:

```
Reduce SSD controller power consumption
while remaining operational
```

States:

```
PS0
PS1
PS2
PS3
PS4
...
```

Each SSD defines its own power states.

Host discovers them via:

```
Identify Controller
```

#### Example NVMe Power States

A drive might advertise:

|State|Power|
|---|---|
|PS0|8 W|
|PS1|5 W|
|PS2|2 W|
|PS3|100 mW|
|PS4|25 mW|

These values vary by SSD.

### What Changes Between NVMe Power States?

The controller may reduce:

```
CPU frequency
DRAM frequency
NAND activity
Background tasks
PCIe activity
```

Example:

```
PS0
  Full performance

PS3
  Reduced clocks
  Reduced power
```

But the SSD is still an operational PCIe device.

### Key Difference

#### PCIe D-State

Controls:

```
Can the PCIe device operate at all?
```

#### NVMe PS-State

Controls:

```
How much power does the NVMe controller consume?
```

##### Example 1

Laptop idle.

```
PCIe Device = D0
NVMe = PS3
```

Meaning:

```
Device fully present
PCIe link works
SSD consuming low power
```

This is extremely common.

##### Example 2

System suspend.

```
PCIe Device = D3hot
NVMe = N/A
```

Since the PCIe device itself is sleeping:

```
NVMe controller no longer operational
```

##### Example 3

Device powered off.

```
PCIe Device = D3cold
```

Then:

```
No NVMe state exists
```

because the controller is not powered.

## Relationship to PCIe Link States

There is a third layer:

```
L0
L0s
L1
L1.1
L1.2
L2
L3
```

These control the PCIe link itself.

Example:

```
PCIe Link = L1.2
PCIe Device = D0
NVMe = PS3
```

This is a very common idle laptop state.

##### Real Laptop Idle Example

```
PCIe Link = L1.2
PCIe Device = D0
NVMe = PS4
```

Meaning:

```
Link sleeping
Device still present
NVMe controller in deep low-power mode
```

Wakeup is quick.

## APST (Autonomous Power State Transition)

NVMe has a feature called:

```
APST
```

which automatically moves between NVMe power states.

Example:

```
Idle 50 ms
    |
    v
PS1

Idle 500 ms
    |
    v
PS3

Idle 5 sec
    |
    v
PS4
```

The PCIe device remains:

```
D0
```

the entire time.

### Common Confusion

Many engineers think:

```
PS4 == D3hot
```

This is incorrect.

Example:

```
D0 + PS4
```

is completely valid. The device is still operational but using very little power.

### Firmware Debug Example

Suppose an SSD disappears after resume.

Check:

##### PCIe Device State

```
D3hot -> D0 transition
```

Did it occur correctly?

##### PCIe Link State

```
L1.2 -> L0
```

Did the link wake?

##### NVMe Power State

```
PS4 -> PS0
```

Did the controller wake? All three layers may be involved.

### Visual Summary

```
PCIe Link Layer

L0
L0s
L1
L1.1
L1.2
L2
L3
```

Independent of:

```
PCIe Device Layer

D0
D1
D2
D3hot
D3cold
```

Independent of:

```
NVMe Controller Layer

PS0
PS1
PS2
PS3
PS4
...
```

## Most Common NVMe SSD Idle State

On modern laptops:

```
PCIe Link = L1.2
PCIe Device = D0
NVMe Power State = PS3 or PS4
```

This gives:

```
Very low power
Fast wakeup
No re-enumeration
```

which is why it is preferred over placing the PCIe device into D3hot/D3cold during ordinary idle periods.

