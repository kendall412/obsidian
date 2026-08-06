In NVMe, **AWUN** and **NAWUN** describe the maximum write size that the device guarantees to be **atomic during normal operation**. Atomic means the write is treated as an indivisible unit: after the write, software should not observe only part of that write updated and part not updated, assuming normal operation.

## AWUN (Atomic Write Unit Normal)

It is a field in the **Identify Controller** data structure. It gives the controller-wide atomic write size for normal operation.

The value is **0-based** and expressed in logical blocks.

So:

```text
atomic write size = AWUN + 1 logical blocks
```

Example:

```text
AWUN = 0
```

means:

```text
1 logical block is atomic
```

Example:

```text
AWUN = 7
```

means:

```text
writes up to 8 logical blocks are atomic
```

---

## NAWUN (Namespace Atomic Write Unit Normal)

It is a field in the **Identify Namespace** data structure. It gives the atomic write size for a specific namespace.

Like AWUN, it is:

```text
0-based
in logical blocks
```

So:

```text
atomic write size = NAWUN + 1 logical blocks
```

Example:

```text
NAWUN = 15
```

means:

```text
writes up to 16 logical blocks are atomic for that namespace
```


## AWUN vs NAWUN

| Field | Location | Scope |
|---|---|---|
| AWUN | Identify Controller | Controller-wide default |
| NAWUN | Identify Namespace | Namespace-specific value |

If the namespace reports valid namespace-specific atomicity fields, then **NAWUN overrides AWUN** for that namespace.

Conceptually:

```text
if namespace-specific atomic fields are valid:
    use NAWUN
else:
    use AWUN
```


### Important: “Normal” does not mean power-fail safe

AWUN/NAWUN apply to **normal operation**. They do **not necessarily guarantee atomicity across power loss**.

For power-fail atomicity, NVMe has separate fields:

| Normal operation | Power-fail condition |
|---|---|
| AWUN | AWUPF |
| NAWUN | NAWUPF |

So if you care about crash or power-loss atomicity, check:

```text
AWUPF / NAWUPF
```

not just:

```text
AWUN / NAWUN
```

### Example

Assume a namespace has:

```text
LBA size = 4096 bytes
NAWUN = 7
```

Then:

```text
atomic unit = NAWUN + 1
            = 8 logical blocks
```

Since each block is 4096 bytes:

```text
8 × 4096 = 32768 bytes
```

So writes up to:

```text
32 KiB
```

are guaranteed atomic during normal operation for that namespace.

## Relationship to NVMe Write command NLB

The NVMe Write command also uses a 0-based block count field called **NLB**.

So if:

```text
NAWUN = 7
```

then a Write command with:

```text
NLB <= 7
```

is within the atomic write unit, because that means:

```text
NLB = 7 → 8 logical blocks
```

## Short summary

- **AWUN** is the controller-level atomic write size during normal operation.
- **NAWUN** is the namespace-specific atomic write size during normal operation.

Both are 0-based:

```text
atomic write size = field value + 1 LBAs
```

Use **NAWUN** when valid for the namespace; otherwise use **AWUN**. For power-loss atomicity, check **AWUPF/NAWUPF** instead.

## Simpler terms

In NVMe, **AWUN** and **NAWUN** tell you:

> “How big of a write can the SSD complete as one all-or-nothing operation during normal operation?”

If the write is within that size, the drive guarantees you won’t see only part of that write completed.


## Simple meaning

### AWUN

**AWUN** = **Atomic Write Unit Normal**

It is the controller-wide value.

Think of it as:

> “For this NVMe controller, writes up to this many blocks are atomic.”

---

### NAWUN

**NAWUN** = **Namespace Atomic Write Unit Normal**

It is the namespace-specific value.

Think of it as:

> “For this particular NVMe namespace, writes up to this many blocks are atomic.”

A namespace is like a logical disk exposed by the NVMe device.

---

## What “atomic” means

Atomic means **all or nothing**.

Example:

Suppose an SSD says writes up to **8 blocks** are atomic.

If you write 8 blocks:

```text
Block 100
Block 101
Block 102
...
Block 107
```

then the device should not end up with only some of those blocks written and others not written during normal operation.

You should see either:

```text
old data in all 8 blocks
```

or:

```text
new data in all 8 blocks
```

Not:

```text
new data in blocks 100-103
old data in blocks 104-107
```


## Important detail: the value is 0-based

AWUN and NAWUN are reported as **one less than the number of blocks**.

So:

```text
actual atomic write size = value + 1
```

Examples:

| Reported value | Actual atomic size |
|---:|---:|
| 0 | 1 block |
| 1 | 2 blocks |
| 7 | 8 blocks |
| 15 | 16 blocks |

So if:

```text
NAWUN = 7
```

that means:

```text
writes up to 8 logical blocks are atomic
```


## Which one should you use?

Usually:

```text
Use NAWUN if the namespace reports it.
Otherwise use AWUN.
```

In plain English:

- **AWUN** is the general/default value for the controller.
- **NAWUN** is the value for one specific namespace.
- Namespace-specific value is more precise.


## Normal operation only

The **N** in AWUN/NAWUN means **Normal**. That means this atomicity applies while the drive is operating normally. It does **not necessarily mean the write is atomic if power is suddenly lost**. The **PF** in AWUPF/NAWUPF means "power failure".

For power-loss atomicity, NVMe has separate fields:

```text
AWUPF
NAWUPF
```

## AWUN/NAWUN vs AWUPF/NAWUPF
|Normal Operation|Power-Fail Condition|
|---|---|
|AWUN|AWUPF|
|NAWUN|NAWUPF|

Usually, the power-fail atomic size may be smaller than the normal atomic size.

```
NAWUN  = 7   → 8 blocks atomic normally
NAWUPF = 0   → 1 block atomic during power loss
```
This means an 8-block write is atomic if the drive is running normally, but if power is lost during that write, only single-block atomicity is guaranteed.


## Tiny example

Assume:

```text
LBA size = 4 KiB
NAWUN = 7
```

Then:

```text
actual atomic size = 7 + 1 = 8 blocks
```

Each block is 4 KiB:

```text
8 × 4 KiB = 32 KiB
```

So this namespace guarantees that writes up to **32 KiB** are atomic during normal operation.


## One-sentence summary

**AWUN/NAWUN tell you the maximum number of logical blocks an NVMe device can write as one all-or-nothing operation during normal operation; AWUN is controller-wide, NAWUN is namespace-specific.**

## AWUN and NAWUN in nvme cli

```bash
sudo nvme id-ctrl /dev/nvme0
```

Look for the awun row in the printed configuration data:
- “awun : 255 awupf : 0 nvscc : 1 acwu : 0”
- Interpretation: The value is zero-based. If awun displays 255, it means 255 + 1 = 266 logical blocks can be written atomically during normal operations. A value of 0 means atomic writes are limited to a single logical block

If you want to see NAWUN (Namespace Atomic Write Unit Normal), which defines the parameter specific to a namespace, use the id-ns command:
```bash
nvme id-ns /dev/nvme0n1
```
