
> The **Controller Status (CSTS)** register is an NVMe controller register that allows the host to determine the **current operational state** of the NVMe controller. It is a **32-bit register** located in the NVMe controller's MMIO register space (typically within PCIe BAR0/BAR1).

The host reads CSTS during:

- Controller initialization
- Controller shutdown
- Controller reset
- Error recovery
- Firmware activation

## Relationship Between CC and CSTS

The host controls the controller through the [[CC (Controller Configuration)]] register.
The controller reports its status through the **Controller Status (CSTS)** register.

Example:

1. Host writes `CC.EN = 1`
2. Controller initializes itself
3. Controller sets `CSTS.RDY = 1`
4. Host knows controller is ready

Similarly:

1. Host writes `CC.EN = 0`
2. Controller stops operation
3. Controller clears `CSTS.RDY = 0`
4. Host knows controller is disabled

## CSTS Register Layout

|Bits|Field|Description|
|---|---|---|
|0|RDY|Ready|
|1|CFS|Controller Fatal Status|
|2|SHST[0]|Shutdown Status|
|3|SHST[1]|Shutdown Status|
|4|NSSRO|NVM Subsystem Reset Occurred|
|31:5|Reserved|Reserved|

## 1. RDY (Ready)

**Bit 0**

Indicates whether the controller is operational.

| Value | Meaning              |
| ----- | -------------------- |
| 0     | Controller not ready |
| 1     | Controller ready     |
### Enable Sequence Example

Host enables controller:

```
CC.EN = 1
```

Controller:

```
Initialize queues
Initialize admin structures
Load firmware
Prepare NAND interface
```

When complete:

```
CSTS.RDY = 1
```

The host must wait until RDY becomes 1 before sending commands.

### Disable Sequence Example

Host disables controller:

```
CC.EN = 0
```

Controller stops processing commands.

When complete:

```
CSTS.RDY = 0
```

## 2. CFS (Controller Fatal Status)

**Bit 1**

Indicates that the controller encountered an unrecoverable internal error.

| Value | Meaning              |
| ----- | -------------------- |
| 0     | No fatal error       |
| 1     | Fatal error detected |

### Examples of Fatal Errors

- Firmware crash
- Internal SRAM corruption
- ECC engine failure
- PCIe internal error
- DMA engine failure
- Unexpected hardware exception

When:

```
CSTS.CFS = 1
```

The controller can no longer reliably process commands.

Usually the host must:

```
Reset controller

or

Power cycle device
```

### Linux Example

(bash)
```
nvme reset /dev/nvme0
```

or (bash)

```
echo 1 > /sys/bus/pci/devices/.../reset
```

## 3. SHST (Shutdown Status)

**Bits [3:2]**

Reports shutdown progress.

### Values

|SHST|Meaning|
|---|---|
|00b|Normal operation|
|01b|Shutdown in progress|
|10b|Shutdown complete|
|11b|Reserved|

### Shutdown Process

Host requests shutdown via CC register:

```
CC.SHN = 01b
```

Controller begins:

```
Flush DRAM cache
Complete metadata updates
Update FTL tables
Persist mapping information
```

During shutdown:

```
CSTS.SHST = 01b
```

After completion:

```
CSTS.SHST = 10b
```

Host can then safely remove power.

### Example Timeline

```
Host              Controller

CC.SHN=01
  ───────────────►

              SHST=01
              Flushing metadata
              Saving FTL

              SHST=10
◄──────────────
Shutdown Complete
```

## 4. NSSRO (NVM Subsystem Reset Occurred)

**Bit 4**

Indicates that an NVM subsystem reset has occurred since the last time the host checked.

|Value|Meaning|
|---|---|
|0|No subsystem reset|
|1|Reset occurred|

### What is an NVM Subsystem Reset?

An NVM subsystem reset resets:

- Controller(s)
- Queues
- Internal state
- Firmware state

without necessarily removing power.

Possible causes:

- Firmware update
- Internal watchdog timeout
- Host-issued subsystem reset
- Error recovery event

When the host sees:

```
CSTS.NSSRO = 1
```

it knows controller state may have changed and reinitialization may be required.

---
## Typical Initialization Flow

### Step 1

Read [[CAP (Controller Capabilities)]] register.

```
CAP
```

Determine:

- Queue sizes
- Timeout values
- Supported features

### Step 2

Program Admin Queues.

```
AQA
ASQ
ACQ
```

### Step 3

Enable controller.

```
CC.EN = 1
```

### Step 4

Poll CSTS.RDY (C).

```
while ((CSTS & 0x1) == 0)
{
    // wait
}
```

### Step 5

Controller becomes ready.

```
CSTS.RDY = 1
```

### Step 6

Issue Admin Commands.

```
Identify
Get Log Page
Create IO Queue
```

## Real Firmware Example

A firmware engineer often sees code similar to (C):

```
/* Enable controller */
writel(cc | NVME_CC_ENABLE, CC_REG);

/* Wait for ready */
timeout = CAP.TO * 500ms;

while (!(readl(CSTS_REG) & NVME_CSTS_RDY))
{
    if (timeout_expired())
        return -ETIMEDOUT;
}
```

This is one of the first interactions between the host driver and the NVMe controller.

## Summary

| Field | Meaning                                |
| ----- | -------------------------------------- |
| RDY   | Controller is ready to accept commands |
| CFS   | Fatal controller error occurred        |
| SHST  | Shutdown progress/status               |
| NSSRO | NVM subsystem reset occurred           |
Think of **CC** and **CSTS** as a request/response pair:

```
Host                     Controller

CC.EN = 1      ─────►

                Initialize

CSTS.RDY = 1   ◄─────

Ready
```

**CC tells the controller what state to enter; CSTS tells the host what state the controller is actually in.** This handshake is fundamental to NVMe initialization, shutdown, reset, and error recovery.

