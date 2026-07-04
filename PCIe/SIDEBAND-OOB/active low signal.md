
>An **active-low sideband signal** is a **dedicated control signal** outside the PCIe data lanes whose **active (asserted) state is a low voltage level (logic 0)** rather than a high voltage level (logic 1).

For example:

- **PERST#**
- **CLKREQ#**
- **PEWAKE#**

are all **active-low sideband signals** in PCIe. The `#` suffix means **active low**.

## Understanding "Active"

"Active" simply means:

> **The signal is currently requesting or performing its function.**

For example, consider a reset signal.

If the signal is active:

```
Reset the device
```

If the signal is inactive:

```
Do not reset the device
```

The question is:

**Which voltage level represents "active"?**

## Active-High vs Active-Low

### Active-High

Logic 1 means active.

```
Voltage

3.3V  ────────────────
        ACTIVE

0V    ________________
        INACTIVE
```

Logic table:

|Voltage|Logic|Meaning|
|---|---|---|
|High|1|Active|
|Low|0|Inactive|
### Active-Low

Logic 0 means active.

```
Voltage

3.3V  ________________
        INACTIVE

0V    ────────────────
        ACTIVE
```

Logic table:

| Voltage | Logic | Meaning  |
| ------- | ----- | -------- |
| High    | 1     | Inactive |
| Low     | 0     | Active   |

## Why the "#" Symbol?

In digital hardware, engineers often append:

```
#
```

or sometimes:

```
_n

/BAR
```

to indicate **active-low**.

Examples:

```
PERST#

CLKREQ#

PEWAKE#
```

All mean:

> "This signal is asserted when driven LOW."

### Example: PERST#

PERST# means:

```
PCI Express Reset
```

When:

```
PERST# = Low
```

the SSD is held in reset.

```
Host
 |
PERST# = 0
 |
SSD

↓

Reset
```

When:

```
PERST# = High
```

the SSD is allowed to operate.

```
Host
 |
PERST# = 1
 |
SSD

↓

Run Normally
```

### Example Timing

```
Time →

PERST#

___________-------------------------

Low         High

Reset      Normal Operation
```

The falling edge asserts reset. The rising edge releases reset.

### Example: CLKREQ\#

CLKREQ# is also active-low.

Normal idle:

```
CLKREQ# = High
```

means:

```
I don't currently need the clock.
```

When the SSD needs the reference clock:

```
CLKREQ# = Low
```

means:

```
Please provide REFCLK.
```

### Example Timing

```
Time →

CLKREQ#

----------------------______________

High                  Low

Clock Not Needed      Need Clock
```

### Example: PEWAKE\#

PEWAKE# is used to wake the platform.

```
PEWAKE# = High

↓

No Wake Request
```

```
PEWAKE# = Low

↓

Wake the System
```

## Why Use Active-Low Signals?

There are several engineering reasons.

### 1. Fail-Safe Design

If a wire breaks:

```
Disconnected
```

the pull-up resistor usually causes the signal to read:

```
High
```

which is the inactive state. This avoids accidentally triggering reset or wake-up.

### 2. Open-Drain Signaling

Many sideband signals use **open-drain/open-collector** outputs.

Example:

```
      +3.3V
        |
     Pull-up
        |
Signal ----+---------
            |
         Device
```

The device can:

- Pull the line LOW.
- Release the line.

It never actively drives the line HIGH. This makes active-low signaling a natural fit.

### 3. Multiple Devices Can Share the Signal

Suppose several devices share a wake signal.

```
SSD1 ----\
           \
SSD2 ------+---- PEWAKE#
           /
SSD3 ----/
```

If any device pulls the line LOW:

```
PEWAKE# = Low
```

the platform wakes. This is known as **wired-AND** (or, in active-low logic, a wired-OR behavior).

## Sideband Signals vs PCIe Data Lanes

PCIe lanes carry:

```
TLPs

DLLPs

Ordered Sets
```

Sideband signals carry only simple control information.

Example:

```
PERST#

0 = Reset

1 = Run
```

No packets are transmitted.

## Typical PCIe Sideband Signals

|Signal|Active When Low?|Function|
|---|---|---|
|PERST#|Yes|Reset device|
|CLKREQ#|Yes|Request reference clock|
|PEWAKE#|Yes|Wake platform|
|PRSNT#|Yes|Card presence detection|

## Real NVMe Startup

```
Power Applied

↓

REFCLK Stable

↓

PERST# = Low

↓

SSD Held in Reset

↓

PERST# = High

↓

LTSSM Starts

↓

Detect

↓

Polling

↓

Configuration

↓

L0
```

Notice:

The SSD begins PCIe initialization only after **PERST# transitions from Low to High**.

# Summary

An **active-low sideband signal** is a dedicated hardware control signal whose **active state is logic 0 (low voltage)**.

In PCIe:

|Signal|Low Means|
|---|---|
|PERST#|Reset the PCIe device|
|CLKREQ#|Request the PCIe reference clock|
|PEWAKE#|Request the platform to wake|
|PRSNT#|Card is present|

The `#` suffix indicates **active-low**, meaning:

- **Low (0)** = **asserted/active**
- **High (1)** = **deasserted/inactive**

This convention is widely used in digital hardware because it works well with pull-up resistors, open-drain outputs, and shared control lines, making it robust for system-level control signals like those used in PCIe.