
PCIe power management exists at **multiple levels**, and this often confuses engineers because there are:

1. **PCIe Link Power States (L0, L0s, L1, L2, L3)**
2. **PCI Device Power States (D0, D1, D2, D3)**
3. **ASPM (Active State Power Management)**
4. **NVMe Power States (PS0, PS1, PS2, etc.)**

Let's focus first on the **PCIe Link Power States**, which are part of the [[Link Training and Status State Machine (LTSSM)]].

## PCIe Link Power States Overview

```
          Active
            |
            v
           L0
          /  \
         /    \
       L0s     L1
                |
                v
               L2
                |
                v
               L3
```

### L0 (Active State)

Normal operation.

```
Link Up
TLPs Flowing
DLLPs Flowing
DMA Active
```

Example:

```
Host <---- PCIe Gen4 x4 ----> NVMe SSD
```

Read/write commands execute normally.

Power consumption is highest.

### L0s (Fast Standby)

L0s is a lightweight power-saving state.

Think:

```
L0s = Link idle for a short time
```

Characteristics:

```
Transmitter partially disabled
Receiver still ready
Very fast wakeup
```

Transition:

```
L0
 |
 v
L0s
 |
 v
L0
```

Wakeup time is very short (typically tens to hundreds of nanoseconds).

### L1 (Deeper Sleep)

Much lower power than L0s.

Characteristics:

```
Transmitters off
Receivers largely inactive
Clocking reduced
```

Transition:

```
L0
 |
 v
L1
 |
 v
L0
```

Wakeup takes longer:

```
Microseconds
```

rather than nanoseconds.

### L1 Substates (Gen3+)

Modern PCIe adds:

```
L1.0
L1.1
L1.2
```

#### L1.0

Basic L1.

```
Link logic retained
Fast recovery
```

#### L1.1

More circuitry powered down.

```
Lower power
Longer wakeup
```

#### L1.2

Deepest common link idle state.

```
Most PHY circuitry off
Reference clocks may stop
Extremely low power
```

Many modern NVMe laptops spend significant time in:

```
PCIe L1.2
```

to save battery.

### L2

Near shutdown state.

Characteristics:

```
Link essentially inactive
Power removal preparation
```

Typically entered during:

```
System shutdown
Device removal
Sleep transitions
```

Transition:

```
L0
 |
 v
L2
```

Returning from L2 generally requires substantial reinitialization.

### L3

Link Off.

```
No PCIe Link
No LTSSM Activity
```

Effectively:

```
Powered off
```

To return:

```
PERST#
Link Training
Enumeration
```

must occur again.

## [[ASPM (Active State Power Management)]]

ASPM automatically moves the link between:

```
L0
L0s
L1
```

based on activity.

Example:

```
L0
 |
No Traffic
 |
 v
L1.2
 |
New Read Command
 |
 v
L0
```

This happens automatically through hardware and firmware cooperation.

### ASPM Modes

Common BIOS settings:

```
Disabled
L0s
L1
L0s + L1
Auto
```

#### ASPM Example with NVMe

Suppose laptop idle:

```
No Reads
No Writes
```

PCIe link:

```
Gen4 x4
```

moves:

```
L0
 |
 v
L1.2
```

Power drops significantly.

User opens a file:

```
SSD asserts activity
```

Link wakes:

```
L1.2
 |
 v
L0
```

Read command executes.

# Device Power States (D States)

Separate from link states.

Defined by PCI power management.

```
D0 = Fully On
D1 = Light Sleep
D2 = Deeper Sleep
D3hot = Nearly Off
D3cold = Power Removed
```

### D0

Normal operation.

```
Controller Running
Link Active
```

### D3hot

Very common.

```
Configuration space retained
Most logic off
```

Device can return without full power cycle.

### D3cold

Deepest device state.

```
Power Removed
```

Equivalent to:

```
Device Off
```

Returning often requires:

```
PERST#
Enumeration
```

again.

## Link State vs Device State

These are independent.

Example:

```
Device = D0
Link   = L1.2
```

means:

```
Device powered
Link sleeping
```

Another example:

```
Device = D3
hotLink= L2
```

means:

```
Device nearly off
Link inactive
```

# NVMe Power States

NVMe also defines controller power states.

Examples:

```
PS0 = Highest Performance
PS1
PS2
PS3
PS4
```

Configured through NVMe commands. These are separate from PCIe power states.

### Example: Laptop Idle

Everything active:

```
PCIe Link = L0
PCI Device = D0
NVMe State = PS0
```

Power high.

After several seconds idle:

```
PCIe Link = L1.2
PCI Device = D0
NVMe State = PS3
```

Power much lower.

#### Example: System Sleep

Entering sleep:

```
L0
 |
 v
L1.2
 |
 v
L2
 |
 v
L3
```

Device may move:

```
D0
 |
 v
D3hot
 |
 v
D3cold
```

# Why Firmware Engineers Care

Power-state bugs often cause:

```
Drive disappears after sleep
Wakeup failure
Unexpected link retraining
High idle power
Random disconnects
```

Typical debugging involves checking:

```
LTSSM state
ASPM settings
L1 Substates
CLKREQ#
PERST#
NVMe APST
```

# Complete Power-State Picture

```
PCIe Link States

L0    Active
 |
 +--> L0s
 |
 +--> L1
       |
       +--> L1.1
       |
       +--> L1.2
 |
 +--> L2
 |
 +--> L3
```

and independently:

```
PCI Device States

D0
D1
D2
D3hot
D3cold
```

and independently:

```
NVMe Power States

PS0
PS1
PS2
PS3
PS4
...
```

# NVMe Engineer's Most Important States

For NVMe SSD debugging, the states you'll encounter most often are:

```
L0      Normal operation
L1.2    Deep PCIe idle power saving
D0      Device active
D3hot   Device sleep
PS0     Maximum performance
PS3/PS4 Low-power NVMe states
```

Many modern laptop NVMe power issues involve interactions between:

```
PCIe ASPM (L1.2)
+
NVMe APST
+
CLKREQ#
```

rather than problems with the NVMe read/write path itself.

