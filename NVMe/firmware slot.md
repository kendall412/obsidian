
A firmware slot is a storage location inside the NVMe controller where a firmware image can be kept. An NVMe SSD may support multiple firmware slots so that it can store more than one firmware version at a time.

For example:

```
Slot 1: factory firmware
Slot 2: currently running firmware
Slot 3: newly downloaded firmware waiting for activation
```

## Why firmware slots exist

Firmware slots allow the drive to:

- keep a backup or factory firmware image
- download a new firmware image without immediately replacing the active one
- activate new firmware after a reset or power cycle
- support safer firmware updates and possible rollback, depending on the device

### Active firmware slot
Only one firmware slot is normally active at a time. The active firmware slot is the slot whose firmware is currently running on the NVMe controller.

Example:

```
Active Firmware Slot:      2
Next Active Firmware Slot: 3
```

This means the controller is currently running the firmware image stored in slot 2.

### Next active firmware slot
NVMe can also report a next active firmware slot.

Example:

```
Active Firmware Slot:      2
Next Active Firmware Slot: 3
```

This means: 
- the controller is currently running firmware from slot 2 - after the next reset, it will activate firmware from slot 3

### How firmware slots are used

A typical NVMe firmware update flow is:

```
1. Host downloads firmware image to the controller
2. Host commits the firmware image to a firmware slot
3. Host requests activation of that slot
4. Controller activates it immediately or after reset
```

The relevant NVMe admin commands are:

- Firmware Image Download
- Firmware Commit

### Where slot information is reported

Firmware slot information is reported in:
```
Get Log Page: Firmware Slot Information Log
Log Identifier: 03h
```

This log shows:

- which slot is active
- which slot will be active after reset
- firmware revision strings for each slot

#### Related Identify Controller field

The NVMe Identify Controller data structure has a field called FRMW that describes firmware update capabilities, such as:

- how many firmware slots are supported
- whether slot 1 is read-only
- whether activation without reset is supported

### Simple summary
A firmware slot in NVMe is like a numbered container for a firmware version on the SSD controller.

```
Slot 1 = firmware version A
Slot 2 = firmware version B
Slot 3 = firmware version C
```

The controller runs from one active slot, and firmware updates are usually written to another slot before being activated.


## How to see FW slot information

use nvme-cli:<br>
`sudo nvme fw-log /dev/nvme0`

Output:
```
Firmware Log for device:nvme0
afi  : 0x21 
frs1 : 80002C00
frs2 : 80003C00
frs3 : 
frs4 : 
```

### How to read it
- `frs1`, `frs2`, etc. = firmware revision stored in each firmware slot
- `afi` = Active Firmware Info

`afi` tells you:
`afi : 0x21` which is `0010 0001`
```
bits 3:0 = active firmware slot (1 or 0001)
bits 7:4 = next active firmware slot (2 or 0010)
```
Example:

`afi : 02h`

means:

```
active slot      = 1
next active slot = 2
```

So the drive is currently running firmware from slot 1, and slot 2 is selected to become active after reset.

### Show controller firmware capabilities
To see how many firmware slots the NVMe device supports:<br>
`sudo nvme id-ctrl /dev/nvme0`<br>
look for: `frmw`<br>
Example:<br>
if `frmw " 16h`<br>
This field tells firmware update capabilities, including the number of supported slots. Some versions of `nvme-cli` decode it if you use verbose output:<br>
`sudo nvme id-ctrl /dev/nvme0 -H`

### Common Commands
```
sudo nvme list
sudo nvme fw-log /dev/nvme0
sudo nvme id-ctrl /dev/nvme0 -H
```
### Typical Workflow
```
sudo nvme list
sudo nvme fw-log /dev/nvme0
sudo nvme id-ctrl /dev/nvme0 -H
```

 ## How Switch FW Slot and Activate it

 You select/activate a different firmware slot using the Firmware Commit command.

With Linux `nvme-cli`, the command is:

```
sudo nvme fw-commit /dev/nvme0 --slot=<slot_number> --action=<action>
```

### Check current firmware slots first
```
sudo nvme fw-log /dev/nvme0
```

Example output may show:

```
afi  : 0x32
frs1 : 1.00A
frs2 : 2.10B
frs3 : 2.20C
```

`afi : 32h` is `0011 0010`
therefore active slot is 2 and next active slot is 3

### Activate an existing firmware slot after reset
To select slot 3 to become active after the next reset:
```
sudo nvme fw-commit /dev/nvme0 --slot=3 --action=3
```

Then reset or reboot:
```
sudo nvme reset /dev/nvme0
```

### Activate an existing firmware slot immediately
If the drive supports immediate activation:
```
sudo nvme fw-commit /dev/nvme0 --slot=3 --action=4
```
Not all NVMe drives support immediate activation.

Example: switch to firmware slot 2
```
sudo nvme fw-commit /dev/nvme0 --slot=2 --action=3
sudo nvme reset /dev/nvme0
```

Then verify:
```
sudo nvme fw-log /dev/nvme0
```

So the key command is:
```
nvme fw-commit /dev/nvme0 --slot=N --action=3
```
---
### Common Action for `fw-commit` command

|Action|Meaning|
|:-:|---|
|`0`|Replace the fw image in the specified fw slot, but do **not activate** it|
|`1`|Replace the fw image in the specified fw slot, and **activate on the next reset**|
|`2`|**Activate** the existing image in the specified slot on the next reset. No new image is committed|
|`3`|Replace the fw image in the specified slot and **activate it immediately** without reset, if supported|
|`4`|**Activate** the existing image in the specified slot immediately, if supported|
|`5`|Replace the specified boot partition contents. Used with `--bpid`; supported only on controllers with boot partitions|
|`6`|Activate the specified boot partition. Used with `--bpid`; supported only on controllers with boot partitions|
|`7`|Reserved|

