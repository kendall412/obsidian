
> **PCIe enumeration** is the process by which a system firmware (BIOS/UEFI) and operating system discover, identify, configure, and allocate resources to all PCIe devices in the system after power-on or reset.


Think of enumeration as the host asking:

> "What PCIe devices are connected, what resources do they need, and how should I configure them?"

# High-Level PCIe Startup Sequence

```
Power On / Reset
       │
       ▼
PCIe Link Training
       │
       ▼
Link Up (LTSSM → L0)
       │
       ▼
PCIe Enumeration
       │
       ▼
BAR Assignment
Interrupt Assignment
Driver Loading
       │
       ▼
Device Ready for Use
```

For an NVMe SSD:

```
Power On
   │
   ▼
PCIe link trains to Gen4 x4
   │
   ▼
Host discovers NVMe controller
   │
   ▼
Assigns BAR memory space
   │
   ▼
Loads NVMe driver
   │
   ▼
Creates Admin Queue
   │
   ▼
SSD operational
```

# Step 1: Link Training Must Complete First

Before enumeration can begin:

- PCIe Root Complex (CPU/chipset)
- PCIe Endpoint (NVMe SSD, GPU, NIC)

must establish a working PCIe link.

The LTSSM (Link Training and Status State Machine) performs:

- Detect
- Polling
- Configuration
- Recovery
- L0

When both sides reach:

```
L0
```


the link is operational.

Example:

```
CPU Root Port
      │
      │ Gen4 x4
      │
NVMe SSD
```

Now the host can access configuration space.


# Step 2: Read PCIe Configuration Space

Every PCIe device contains a mandatory:

```
4 KB Configuration Space
```

accessible through Configuration Transactions.

The host starts by probing:

```
Bus 0 Device 0 Function 0
```

and continues scanning.

# Configuration Space Header

First 64 bytes contain:

```
Offset  Field
------  -----------------
0x00    Vendor ID
0x02    Device ID
0x04    Command Register
0x06    Status Register
0x08    Revision ID
0x09    Class Code
0x10    BAR0
0x14    BAR1
...
```

Example:

```
Vendor ID = 8086h
Device ID = F1A5h
```

Host immediately knows:

```
Intel device
```


# Step 3: Discover Devices

Host scans all possible locations:

```
Bus Number
Device Number
Function Number
```

Known as:

```
BDF
```

(Bus:Device.Function)

Example:

```
0000:01:00.0
```

means:

```
Bus      = 1
Device   = 0
Function = 0
```

Linux example (bash):

```
lspci
```

Output:

```
01:00.0 Non-Volatile memory controller
```

# Step 4: Discover PCIe Switches and Bridges

A PCIe topology may contain switches.

Example:

```
Root Complex
      │
      ▼
 PCIe Switch
   ├── NVMe SSD
   ├── NIC
   └── GPU
```

The host detects bridges and assigns:

```
Primary Bus Number
Secondary Bus Number
Subordinate Bus Number
```

This allows recursive enumeration.

Example:

```
Root Port
  Bus 0

Switch Downstream Port
  Bus 1

GPU
  Bus 2
```

# Step 5: Determine BAR Requirements

Every PCIe device requests memory resources through:

```
Base Address Registers (BARs)
```

Examples:

| Device   | BAR Usage            |
| -------- | -------------------- |
| NVMe SSD | Controller Registers |
| GPU      | Frame Buffer         |
| NIC      | Device Registers     |
## How Host Determines BAR Size

Host writes:

```
FFFFFFFFh
```

to BAR.

Device responds with size mask.

Example:

```
Write: FFFFFFFFh
Read: FFFF0000h
```

Host calculates:

```
64 KB BAR
```

required.


# Step 6: Assign Memory Addresses

Host allocates MMIO address ranges.

Example:

```
BAR0 = 0x90000000
```

NVMe controller registers become:

```
0x90000000 + offset
```

Example:

```
CAP register
```

at offset 0x0:

```
0x90000000
```

Doorbell register:

```
0x90001000
```

# Step 7: Enable Device

Host sets bits in the Command Register.

```
PCI Command Register
```

Important bits:

|Bit|Meaning|
|---|---|
|0|Memory Space Enable|
|1|I/O Space Enable|
|2|Bus Master Enable|

Example:

```
Memory Enable = 1
Bus Master = 1
```

Now device can:

- respond to MMIO accesses
- initiate DMA transfers

# Step 8: Configure Interrupts

Modern PCIe devices use:

```
MSIMSI-X
```

instead of legacy INTx.

Host:

1. Reads MSI/MSI-X capability
2. Allocates interrupt vectors
3. Programs MSI tables

Example:

```
NVMe Queue 1 → Vector 32
NVMe Queue 2 → Vector 33
```


# Step 9: Driver Loads

Using:

```
Vendor ID
Device ID
Class Code
```

OS chooses a driver.

For NVMe:

```
Class Code = 01h
Subclass   = 08h
```

meaning:

```
Mass Storage
Non-Volatile Memory
```

OS loads the NVMe driver.

# Step 10: Device-Specific Initialization

For NVMe SSD:

Driver:

1. Maps BAR0
2. Reads CAP register
3. Configures Admin SQ
4. Configures Admin CQ
5. Writes ASQ/ACQ
6. Sets CC.EN
7. Waits for CSTS.RDY

After:

```
CSTS.RDY = 1
```

controller is ready.

# Example Enumeration of an NVMe SSD

```
CPU
 │
 ▼
Root Port
 │
 ▼
NVMe SSD
```

### Host actions

### 1. Link Training

```
Gen4 x4
L0 state reached
```

### 2. Read Config Space

```
Vendor ID = 144D
Device ID = A808
```

(Samsung NVMe)

### 3. Determine BAR

```
BAR0 size = 16 KB
```

### 4. Assign MMIO

```
BAR0 = 0xA0000000
```

### 5. Enable Bus Master

```
Command Register:
Memory Enable = 1
Bus Master = 1
```

### 6. Configure MSI-X

```
64 interrupt vectors
```

### 7. Load NVMe Driver

```
nvme.sys (Windows)
nvme.ko (Linux)
```

### 8. Initialize Controller

```
Create Admin QueuesEnable Controller
```

### 9. Enumerate Namespaces

```
Namespace 1 found
```

Drive becomes visible.

# Why PCIe Enumeration Is Critical for NVMe

Without enumeration:

- BAR addresses are unknown
- DMA cannot work
- MSI-X interrupts cannot work
- Driver cannot communicate with controller

Enumeration provides:

```
Device Discovery
Resource Allocation
Address Assignment
Interrupt Configuration
Driver Binding
```

Everything that happens before the first NVMe Admin command is sent.

In a typical NVMe boot sequence:

```
Power On
  ↓
PCIe Link Training
  ↓
PCIe Enumeration
  ↓
BAR Assignment
  ↓
NVMe Driver Loads
  ↓
Admin Queue Creation
  ↓
Identify Controller
  ↓
Identify Namespace
  ↓
SSD Ready
```

This is the complete path from power-on to a usable NVMe device.