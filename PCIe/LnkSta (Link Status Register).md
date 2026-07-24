
> The **Link Status Register (LNKSTA)** is a register in the PCIe Capability structure that reports the **current operational state of the PCIe link**.

It tells software things like:

- Current link speed
- Current link width
- Whether link training is occurring
- Whether there was a bandwidth change event

For an NVMe SSD, this is one of the most useful registers when debugging PCIe link issues.

# Where is the Link Status Register?

PCIe Configuration Space contains a PCI Express Capability structure:

```
PCIe Capability

+----------------------+
| PCIe Capabilities    |
+----------------------+
| Device Capabilities  |
+----------------------+
| Device Control       |
+----------------------+
| Device Status        |
+----------------------+
| Link Capabilities    |
+----------------------+
| Link Control         |
+----------------------+
| Link Status          |
+----------------------+
```

The Link Status Register is a **16-bit register**.

# Link Control and Link Status

These two registers are often paired:

```
Link Control (LNKCTL)
        |
        v
Controls link behavior

Link Status (LNKSTA)
        |
        v
Reports current link state
```

Think of:

```
LNKCTL = Commands
LNKSTA = Results
```

# Link Status Register Format

The exact interpretation depends on PCIe generation, but the classic layout is:

```
15                                  0
+----+----+-----+------+-----------+
|LABS|LBMS|LT   |NLW   |CLS        |
+----+----+-----+------+-----------+
```

Important fields:

| Field | Meaning                          |
| ----- | -------------------------------- |
| CLS   | Current Link Speed               |
| NLW   | Negotiated Link Width            |
| LT    | Link Training                    |
| LBMS  | Link Bandwidth Management Status |
| LABS  | Link Autonomous Bandwidth Status |
# Current Link Speed (CLS)

Reports the currently negotiated speed.

Values:

| Value | PCIe Speed      |
| ----- | --------------- |
| 1     | Gen1 (2.5 GT/s) |
| 2     | Gen2 (5.0 GT/s) |
| 3     | Gen3 (8.0 GT/s) |
| 4     | Gen4 (16 GT/s)  |
| 5     | Gen5 (32 GT/s)  |
| 6     | Gen6 (64 GT/s)  |
Example:

```
CLS = 4
```

means:

```
Current Speed = Gen4
```

# Negotiated Link Width (NLW)

Reports how many lanes are currently active.

Values:

```
1  -> x1
2  -> x2
4  -> x4
8  -> x8
16 -> x16
```

Example:

```
NLW = 4
```

means:

```
Current Width = x4
```

# Link Training Bit (LT)

Indicates whether the link is currently training.

### LT = 0

```
Training complete
Link operational
```

### LT = 1

```
Link training in progress
```

Occurs during:

```
Reset
Hot reset
Recovery
Speed change
```

# Link Bandwidth Management Status (LBMS)

Indicates that the link bandwidth changed.

Example:

```
Gen4 x4
      |
      v
Gen3 x4
```

The bit is set so software knows the link changed.

# Link Autonomous Bandwidth Status (LABS)

Indicates an autonomous bandwidth change.

Example:

```
Device automatically changed
speed or width
```

without explicit software control.

# Example: Healthy Gen4 NVMe SSD

Suppose an M.2 SSD is operating normally.

Link Status may report:

```
Current Link Speed      = Gen4
Negotiated Link Width   = x4
Link Training           = 0
```

Meaning:

```
PCIe Gen4 x4
Link fully operational
```

# Example: Width Downtraining

Expected:

```
Gen4 x4
```

Actual:

```
Current Speed = Gen4
Width         = x2
```

Link Status shows:

```
CLS = 4
NLW = 2
```

Meaning:

```
Link trained at x2 instead of x4
```

Possible causes:

- Damaged lane
- Signal integrity problem
- Connector issue

# Example: Speed Downtraining

Expected:

```
Gen5 x4
```

Actual:

```
Gen3 x4
```

Link Status:

```
CLS = 3
NLW = 4
```

Meaning:

```
Speed degradedWidth OK
```

Possible causes:

- Equalization failure
- PHY issue
- Poor signal quality

# Linux Example

You can view Link Status with (bash):

```
lspci -vv
```

Example output:

```
LnkSta:
    Speed 16GT/s
    Width x4
```

Meaning:

```
Current Link Speed = Gen4
Negotiated Width   = x4
```

For an NVMe SSD, you might see:

```
01:00.0 Non-Volatile memory controller
```

ollowed by:

```
LnkSta:
    Speed 16GT/s
    Width x4
```

# Difference Between Link Capabilities and Link Status

### Link Capabilities Register (LNKCAP)

Reports what the device supports.

Example:

```
Max Speed = Gen5
Max Width = x4
```

### Link Status Register (LNKSTA)

Reports what is currently active.

Example:

```
Current Speed = Gen4
Current Width = x2
```

So:

```
Supported != Active
```

A device capable of Gen5 x4 may actually be running at Gen4 x2.

# NVMe Debugging Example

Suppose an SSD should train as:

```
Gen4 x4
```

Read Link Status:

```
Speed = Gen1
Width = x1
```

This immediately tells you:

```
LTSSM training succeeded
but link negotiated poorly
```

and directs debugging toward:

- PHY
- Equalization
- Signal integrity
- Lane detection

rather than NVMe command processing.

# Summary

The **Link Status Register (LNKSTA)** is a PCIe configuration-space register that reports the current state of the PCIe link.

Important fields:

| Field                       | Purpose                       |
| --------------------------- | ----------------------------- |
| Current Link Speed (CLS)    | Active PCIe generation        |
| Negotiated Link Width (NLW) | Active lane count             |
| Link Training (LT)          | Whether training is occurring |
| LBMS                        | Bandwidth change detected     |
| LABS                        | Autonomous bandwidth change   |

For NVMe SSDs, the most commonly checked values are:

```
Current Link Speed
Negotiated Link Width
```

A healthy PCIe Gen4 NVMe SSD typically reports:

```
Speed = Gen4 (16 GT/s)
Width = x4
Link Training = 0
```

indicating that LTSSM completed successfully and the drive is operating at full PCIe bandwidth.