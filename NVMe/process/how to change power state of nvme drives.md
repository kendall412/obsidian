
> In NVMe, changing the power state is done by sending the **Set Features** Admin command with the **Power Management feature (Feature Identifier = 0x02)**.

# High-Level Flow

```
Host
  ↓
Identify Controller
  ↓
Discover supported power states
  ↓
Set Features (FID = 0x02)
  ↓
Controller validates request
  ↓
Controller transitions to new power state
  ↓
Completion returned
```

# 1. Discover Supported Power States

Before changing power states, the host reads the controller's Identify data structure.

Admin Command:

```
Identify Controller
Opcode = 06h
```

The returned Identify Controller structure contains:

```
NPSS = Number of Power States Supported
```

and a table:

```
Power State Descriptor 0
Power State Descriptor 1
Power State Descriptor 2
...
Power State Descriptor N
```

Example:

|Power State|Power Consumption|
|---|---|
|PS0|8 W|
|PS1|5 W|
|PS2|3 W|
|PS3|1 W|
|PS4|50 mW|

Typically:

```
PS0 = highest performance
PSn = lowest power
```

# 2. Examine Power State Descriptors

Each descriptor contains:

```
MP   = Maximum Power
ENLAT = Entry Latency
EXLAT = Exit Latency
RRT/RRL = Relative Read Throughput
RWT/RWL = Relative Write Throughput
```

Example:

```
PS0
  MP = 8W
  ENLAT = 0
  EXLAT = 0

PS3
  MP = 1W
  ENLAT = 1000 us
  EXLAT = 5000 us
```

Meaning:

```
Entering PS3 takes 1 ms
Exiting PS3 takes 5 ms
```

# 3. Send Set Features Command

Admin Command:

```
Opcode = 09h
```

Feature Identifier:

```
FID = 02h(Power Management)
```

The requested power state is placed in:

```
CDW11[4:0]
```

Example:

```
Set Features  FID = 02h  Power State = 3
```

Request:

```
Transition controller to PS3
```

# SQ Entry Example

```
DW0:
  OPC = 09h

DW10:
  FID = 02h

DW11:
  PS = 3
```

# 4. Controller Validates Request

The controller checks:

```
Requested PS exists?
Requested PS supported?
Controller ready?
Current operation allows transition?
```

Examples of failures:

```
Invalid Power State
Feature Not Changeable
Invalid Field in Command
```

# 5. Controller Performs Transition

The controller may:

```
Reduce NAND clock
Reduce controller clock
Disable CPU cores
Disable channels
Disable caches
Reduce PCIe activity
```

Example:

```
PS0
  CPU = 800 MHz

PS3
  CPU = 100 MHz
```

The exact implementation is vendor-specific.

# 6. Completion Returned

If successful:

```
SCT = 0
SC  = 0
```

Completion Queue entry:

```
Successful Completion
```

---
# Example Using Linux nvme-cli

Show supported power states (bash):

```
sudo nvme id-ctrl /dev/nvme0
```

Look for:

```
npss : 4

ps 0:
ps 1:
ps 2:
ps 3:
ps 4:
```

Set power state to PS3 (bash):

```
sudo nvme set-feature /dev/nvme0 -f 2 -V 3
```

Where:

```
-f 2  = Power Management feature
-V 3  = Power State 3
```

---

Read current setting (bash):

```
sudo nvme get-feature /dev/nvme0 -f 2
```

Example output:

```
Current value: 00000003
```

Meaning:

```
Current power state = PS3
```

# [[APST (Autonomous Power State Transitions)]]

Modern NVMe SSDs usually do not rely solely on manual power-state changes.

NVMe supports:

```
Autonomous Power State Transition (APST)
```

Feature:

```
FID = 0x0C
```

With APST enabled:

```
Drive automatically transitions:

PS0
 ↓
PS1
 ↓
PS2
 ↓
PS3
```

after specified idle times.

Example:

```
Idle 50 ms → PS1
Idle 500 ms → PS2
Idle 5 sec → PS3
```

No host intervention required after configuration.

# Relationship to PCIe Power Management

Do not confuse NVMe power states with PCIe link power states.

NVMe Power States:

```
PS0
PS1
PS2
PS3
...
```

Control:

```
SSD internal power consumption
```

PCIe Link States:

```
L0
L0s
L1
L1.1
L1.2
```

Control:

```
PCIe link power consumption
```

Both mechanisms can operate simultaneously.


# Example Timeline

```
Host sends Set Features (FID=02, PS=3)
        ↓
Command placed in Admin SQ
        ↓
Host rings SQ doorbell
        ↓
Controller fetches command
        ↓
Controller validates PS3 exists
        ↓
Controller reduces clocks/power
        ↓
Controller enters PS3
        ↓
Completion posted to CQ
        ↓
Host receives success status
```

> The most common method in operating systems is not manual switching to PS3/PS4; instead, the OS configures **APST (FID 0x0C)** and lets the NVMe controller automatically move among supported power states based on idle time.

