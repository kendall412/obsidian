## PCIe Configuration Space: Overview

PCI Express devices expose a standardized **configuration space** that software/firmware uses to discover, identify, configure, and manage devices.

PCIe configuration space is an evolution of PCI configuration space:

| Type | Size |
|---|---:|
| Conventional PCI configuration space | 256 bytes |
| PCIe extended configuration space | 4096 bytes |

<!-- tags: #function -->
Every PCIe [[function]] has its own 4 KiB configuration space.

A PCIe device may contain:

- One function, function `0`
- Multiple functions, functions `0–7`
- Multiple PCIe devices behind bridges or switches

Each function is identified by a **Bus:Device:Function**, often written as:

```text
BB:DD.F
```

Example:

```text
03:00.0
```

means:

- Bus `03`
- Device `00`
- Function `0`

---

# 1. PCIe Configuration Space Layout

A PCIe function’s configuration space is divided into several major areas:

```text
Offset Range      Size        Description
------------------------------------------------------------
0x000–0x03F       64 bytes    Standard PCI configuration header
0x040–0x0FF       192 bytes   PCI capabilities list
0x100–0xFFF       3840 bytes  PCIe extended capabilities
```

Visual layout:

```text
+-------------------------------+ 0x000
| Standard PCI Header           |
| Type 0 or Type 1              |
+-------------------------------+ 0x040
| PCI Capability Structures     |
| MSI, MSI-X, PCIe Capability,  |
| Power Management, etc.        |
+-------------------------------+ 0x100
| PCIe Extended Capabilities    |
| AER, ACS, ARI, SR-IOV, ATS,   |
| Resizable BAR, etc.           |
+-------------------------------+ 0xFFF
```

---

# 2. Standard PCI Configuration Header

The first 64 bytes are common to both PCI and PCIe devices.

There are multiple header types, but the most common are:

| Header Type | Used By |
|---|---|
| Type 0 | Endpoint devices |
| Type 1 | PCI-to-PCI bridges, including PCIe Root Ports and Switch Ports |
| Type 2 | CardBus bridges, mostly obsolete |

---

# 3. Common Header Fields

The first 16 bytes are common across header types.

```text
Offset   Size   Field
-------------------------------------------------
0x00     16     Vendor ID
0x02     16     Device ID
0x04     16     Command
0x06     16     Status
0x08      8     Revision ID
0x09      8     Programming Interface
0x0A      8     Subclass
0x0B      8     Base Class
0x0C      8     Cache Line Size
0x0D      8     Latency Timer
0x0E      8     Header Type
0x0F      8     BIST
```


```
lspci -s 02:00.0 -xxxx

     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f  
00: 5c 1c 70 2b 06 04 10 00 00 02 08 01 10 00 00 00

Vendor ID: 5c 1c
Device ID: 70 2b
Command: 06 04
Rev ID: 00
Prog. Interface: 02
Subclass: 08
Base Class: 01
Cache Line Sz: 10
Latency Timer: 00
Header Type: 00
BIST: 00
```



---

## 3.1 Vendor ID

Offset:

```text
0x00
```

Size:

```text
16 bits
```

Identifies the vendor.

Examples:

| Vendor | Vendor ID |
|---|---:|
| Intel | `0x8086` |
| AMD | `0x1022` |
| NVIDIA | `0x10DE` |
| Broadcom | `0x14E4` |
| Red Hat/QEMU | `0x1AF4` |

A value of:

```text
0xFFFF
```

means no device is present at that Bus:Device:Function.

---

## 3.2 Device ID

Offset:

```text
0x02
```

Size:

```text
16 bits
```

Identifies the specific device model assigned by the vendor.

The pair:

```text
Vendor ID : Device ID
```

uniquely identifies the device type for driver matching.

---

## 3.3 Command Register

Offset:

```text
0x04
```

Size:

```text
16 bits
```

Controls basic device behavior.

Common bits:

| Bit | Name | Meaning |
|---:|---|---|
| 0 | I/O Space Enable | Allows device to respond to I/O port BARs |
| 1 | Memory Space Enable | Allows device to respond to memory BARs |
| 2 | Bus Master Enable | Allows device to initiate DMA |
| 6 | Parity Error Response | Legacy parity handling |
| 8 | SERR# Enable | Enables system error reporting |
| 10 | Interrupt Disable | Disables legacy INTx interrupts |

Important bits for drivers:

```text
Memory Space Enable
Bus Master Enable
Interrupt Disable
```

A device usually cannot perform DMA unless **Bus Master Enable** is set.

---

## 3.4 Status Register

Offset:

```text
0x06
```

Size:

```text
16 bits
```

Reports device status and capability-list support.

Important bits:

| Bit | Name | Meaning |
|---:|---|---|
| 3 | Interrupt Status | Legacy INTx interrupt pending |
| 4 | Capabilities List | Device implements capabilities at offset given by Capabilities Pointer |
| 5 | 66 MHz Capable | Legacy PCI |
| 7 | Fast Back-to-Back Capable | Legacy PCI |
| 8 | Master Data Parity Error | Legacy PCI |
| 11 | Signaled Target Abort | Legacy PCI |
| 12 | Received Target Abort | Legacy PCI |
| 13 | Received Master Abort | Legacy PCI |
| 14 | Signaled System Error | Legacy PCI |
| 15 | Detected Parity Error | Legacy PCI |

For PCIe devices, bit 4 is usually set, indicating that the capability list exists.

---

## 3.5 Class Code

Offsets:

```text
0x09–0x0B
```

The class code identifies the general type of device.

It is composed of:

```text
Base Class
Subclass
Programming Interface
```

Example:

```text
0x01 0x06 0x01
```

means:

```text
Mass Storage Controller / SATA Controller / AHCI
```

Common base classes:

| Base Class | Meaning |
|---:|---|
| `0x01` | Mass Storage Controller |
| `0x02` | Network Controller |
| `0x03` | Display Controller |
| `0x04` | Multimedia Controller |
| `0x06` | Bridge Device |
| `0x0C` | Serial Bus Controller |

---

## 3.6 Header Type

Offset:

```text
0x0E
```

Size:

```text
8 bits
```

Important bits:

| Bits | Meaning |
|---|---|
| 6:0 | Header layout type |
| 7 | Multifunction device |

Examples:

| Value | Meaning |
|---:|---|
| `0x00` | Type 0 endpoint |
| `0x01` | Type 1 bridge |
| `0x80` | Multifunction Type 0 endpoint |
| `0x81` | Multifunction Type 1 bridge |

---

# 4. Type 0 Header: Endpoint Device

A **Type 0 header** is used by PCI/PCIe endpoint functions such as:

- Network cards
- NVMe controllers
- GPUs
- USB controllers
- SATA controllers

Layout:

```text
Offset   Size   Field
-------------------------------------------------
0x00     16     Vendor ID
0x02     16     Device ID
0x04     16     Command
0x06     16     Status
0x08      8     Revision ID
0x09      8     Programming Interface
0x0A      8     Subclass
0x0B      8     Base Class
0x0C      8     Cache Line Size
0x0D      8     Latency Timer
0x0E      8     Header Type
0x0F      8     BIST
0x10     32     BAR0
0x14     32     BAR1
0x18     32     BAR2
0x1C     32     BAR3
0x20     32     BAR4
0x24     32     BAR5
0x28     32     CardBus CIS Pointer
0x2C     16     Subsystem Vendor ID
0x2E     16     Subsystem ID
0x30     32     Expansion ROM Base Address
0x34      8     Capabilities Pointer
0x35     24     Reserved
0x38     32     Reserved
0x3C      8     Interrupt Line
0x3D      8     Interrupt Pin
0x3E      8     Min_Gnt
0x3F      8     Max_Lat
```

```
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
10: 04 00 50 75 00 00 00 00 00 00 00 00 00 00 00 00
20: 00 00 00 00 00 00 00 00 00 00 00 00 5c 1c 5c 1c
30: 00 00 00 00 40 00 00 00 00 00 00 00 ff 01 00 00

BARO: 04 00 50 75
BAR1: 00 00 00 00 
BAR2: 00 00 00 00
BAR3: 00 00 00 00 
BAR4: 00 00 00 00 
BAR5: 00 00 00 00 
CardBus CIS Pointer: 00 00 00 00
Subsystem Vendor ID: 5c 1c
Subsystem ID: 5c 1c
Expansion ROM Base Address: 00 00 00 00
Capabilities Pointer: 00

```


---

## 4.1 Base Address Registers, BARs

Offsets:

```text
0x10–0x24
```

A Type 0 endpoint has up to six BARs.

BARs describe memory or I/O regions used by the device.

A BAR can be:

- Memory BAR
- I/O BAR
- 32-bit BAR
- 64-bit BAR
- Prefetchable or non-prefetchable

Memory BAR format:

```text
Bit 0      0 = Memory BAR
Bits 2:1   Type
           00 = 32-bit address
           10 = 64-bit address
Bit 3      Prefetchable
Bits 31:4  Base address
```

I/O BAR format:

```text
Bit 0      1 = I/O BAR
Bits 1     Reserved
Bits 31:2  Base address
```

For a 64-bit memory BAR, two adjacent BAR registers are consumed:

```text
BAR0 = lower 32 bits
BAR1 = upper 32 bits
```

So a 64-bit BAR at BAR0 uses both BAR0 and BAR1.

---

## 4.2 Determining BAR Size

Software determines BAR size using this classic method:

1. Save original BAR value
2. Write all 1s, `0xFFFFFFFF`, to the BAR
3. Read back the value
4. Mask off attribute bits
5. Invert and add 1
6. Restore original BAR value

Example for a 32-bit memory BAR:

```text
size = ~(read_value & 0xFFFFFFF0) + 1
```

For a 64-bit BAR, both lower and upper registers are used.

---

## 4.3 Subsystem Vendor ID and Subsystem ID

Offsets:

```text
0x2C–0x2F
```

These identify the board or subsystem implementation.

For example, two devices may have the same controller chip but different board vendors.

Drivers sometimes use these IDs for quirks or board-specific behavior.

---

## 4.4 Expansion ROM Base Address

Offset:

```text
0x30
```

Used to map an optional expansion ROM.

Common for:

- GPUs
- Network cards
- RAID controllers

The least significant bit enables or disables the ROM mapping.

---

## 4.5 Interrupt Line and Interrupt Pin

Offsets:

```text
0x3C–0x3D
```

Legacy interrupt support.

| Field | Meaning |
|---|---|
| Interrupt Line | Legacy IRQ routing value, often obsolete on modern PCIe |
| Interrupt Pin | INTA#, INTB#, INTC#, INTD# |

Modern PCIe devices usually use:

- MSI
- MSI-X

instead of legacy INTx interrupts.

---

# 5. Type 1 Header: Bridge Device

A **Type 1 header** is used by bridges, including:

- PCIe Root Ports
- PCIe Switch Upstream Ports
- PCIe Switch Downstream Ports
- PCI-to-PCI bridges

Layout:

```text
Offset   Size   Field
-------------------------------------------------
0x00     16     Vendor ID
0x02     16     Device ID
0x04     16     Command
0x06     16     Status
0x08      8     Revision ID
0x09      8     Programming Interface
0x0A      8     Subclass
0x0B      8     Base Class
0x0C      8     Cache Line Size
0x0D      8     Latency Timer
0x0E      8     Header Type
0x0F      8     BIST
0x10     32     BAR0
0x14     32     BAR1
0x18      8     Primary Bus Number
0x19      8     Secondary Bus Number
0x1A      8     Subordinate Bus Number
0x1B      8     Secondary Latency Timer
0x1C      8     I/O Base
0x1D      8     I/O Limit
0x1E     16     Secondary Status
0x20     16     Memory Base
0x22     16     Memory Limit
0x24     16     Prefetchable Memory Base
0x26     16     Prefetchable Memory Limit
0x28     32     Prefetchable Base Upper 32 Bits
0x2C     32     Prefetchable Limit Upper 32 Bits
0x30     16     I/O Base Upper 16 Bits
0x32     16     I/O Limit Upper 16 Bits
0x34      8     Capabilities Pointer
0x35     24     Reserved
0x38     32     Expansion ROM Base Address
0x3C      8     Interrupt Line
0x3D      8     Interrupt Pin
0x3E     16     Bridge Control
```

---

## 5.1 Bridge Bus Numbers

Important fields:

```text
Primary Bus Number
Secondary Bus Number
Subordinate Bus Number
```

Example:

```text
Primary:     00
Secondary:   03
Subordinate: 05
```

Meaning:

- The bridge is on bus `00`
- Devices behind the bridge start on bus `03`
- The bridge forwards config transactions up to bus `05`

These fields are essential for PCI/PCIe enumeration.

---

## 5.2 Bridge Address Windows

A bridge forwards transactions to devices behind it only if they fall into configured address windows.

Main bridge windows:

| Window | Fields |
|---|---|
| I/O space | I/O Base, I/O Limit |
| Non-prefetchable memory | Memory Base, Memory Limit |
| Prefetchable memory | Prefetchable Memory Base/Limit |

For PCIe, memory windows are especially important.

If a device behind a bridge has a BAR assigned outside the bridge’s memory window, accesses will not reach the device.

---

# 6. PCI Capabilities List

The PCI capabilities list lives within the first 256 bytes of config space, usually starting from an offset given by the **Capabilities Pointer** at `0x34`.

This list exists if the Status register has bit 4 set:

```text
Status[4] = Capabilities List
```

Each capability has a standard header:

```text
Offset +0   Capability ID
Offset +1   Next Capability Pointer
Offset +2   Capability-specific data
```

Structure:

```c
struct pci_capability_header {
    uint8_t cap_id;
    uint8_t next;
};
```

The list is a linked list:

```text
Capabilities Pointer -> Capability -> Capability -> ... -> 0x00
```

Example:

```text
0x34: 0x40

0x40: Capability ID = 0x01, Next = 0x50
0x50: Capability ID = 0x05, Next = 0x70
0x70: Capability ID = 0x10, Next = 0x00
```

---

## 6.1 Common PCI Capability IDs

| ID | Capability |
|---:|---|
| `0x01` | Power Management |
| `0x05` | MSI |
| `0x10` | PCI Express Capability |
| `0x11` | MSI-X |
| `0x13` | PCI Advanced Features |
| `0x14` | Enhanced Allocation |
| `0x09` | Vendor-Specific Capability |

---

# 7. PCI Express Capability Structure

The PCI Express Capability is one of the most important capabilities.

Capability ID:

```
0x10
```

It is located somewhere in the conventional capabilities list, not at a fixed offset.

Main fields include:

```text
Offset from PCIe Capability Base
-------------------------------------------------
0x00   Capability ID
0x01   Next Capability Pointer
0x02   PCI Express Capabilities Register
0x04   Device Capabilities
0x08   Device Control
0x0A   Device Status
0x0C   Link Capabilities
0x10   Link Control
0x12   Link Status
0x14   Slot Capabilities
0x18   Slot Control
0x1A   Slot Status
0x1C   Root Control
0x1E   Root Capabilities
0x20   Root Status
0x24   Device Capabilities 2
0x28   Device Control 2
0x2A   Device Status 2
0x2C   Link Capabilities 2
0x30   Link Control 2
0x32   Link Status 2
```

Not all fields are valid for all device/port types.

---

## 7.1 PCI Express Capabilities Register

This field identifies the device/port type.

Important bits include:

| Field | Meaning |
|---|---|
| Capability Version | PCIe capability version |
| Device/Port Type | Endpoint, Root Port, Switch Port, etc. |
| Slot Implemented | Indicates slot-related registers exist |
| Interrupt Message Number | MSI/MSI-X vector used for PCIe events |

Common PCIe device/port types:

| Type | Meaning |
|---:|---|
| `0x0` | PCIe Endpoint |
| `0x1` | Legacy PCIe Endpoint |
| `0x4` | Root Port |
| `0x5` | Upstream Port of PCIe Switch |
| `0x6` | Downstream Port of PCIe Switch |
| `0x7` | PCIe-to-PCI/PCI-X Bridge |
| `0x8` | PCI/PCI-X-to-PCIe Bridge |
| `0x9` | Root Complex Integrated Endpoint |
| `0xA` | Root Complex Event Collector |

---

## 7.2 Device Capabilities and Device Control

These registers describe and control endpoint/device behavior.

Device Capabilities include things like:

- Maximum payload size supported
- Phantom functions support
- Extended tag support
- Endpoint L0s/L1 acceptable latency
- Role-based error reporting
- Function-level reset support

Device Control includes enables and settings such as:

| Field | Meaning |
|---|---|
| Correctable Error Reporting Enable | Enables reporting of correctable errors |
| Non-Fatal Error Reporting Enable | Enables non-fatal error reporting |
| Fatal Error Reporting Enable | Enables fatal error reporting |
| Unsupported Request Reporting Enable | Enables unsupported request reporting |
| Relaxed Ordering Enable | Allows relaxed ordering |
| Max Payload Size | Configures transaction payload size |
| Extended Tag Enable | Enables larger tag field |
| No Snoop Enable | Allows no-snoop transactions |
| Max Read Request Size | Configures max read request size |
| Bridge Configuration Retry Enable | Bridge-related |
| Initiate Function Level Reset | Requests FLR if supported |

---

## 7.3 Link Capabilities, Link Control, Link Status

These describe the PCIe link.

Important Link Status fields:

| Field | Meaning |
|---|---|
| Current Link Speed | Actual negotiated speed |
| Negotiated Link Width | Actual negotiated width |
| Link Training | Whether link training is in progress |
| Slot Clock Configuration | Same reference clock indication |
| Data Link Layer Link Active | Link is active |

Common link speeds:

| Encoded Value | Speed |
|---:|---|
| `1` | 2.5 GT/s, PCIe Gen1 |
| `2` | 5.0 GT/s, PCIe Gen2 |
| `3` | 8.0 GT/s, PCIe Gen3 |
| `4` | 16.0 GT/s, PCIe Gen4 |
| `5` | 32.0 GT/s, PCIe Gen5 |
| `6` | 64.0 GT/s, PCIe Gen6 |

Common link widths:

```text
x1, x2, x4, x8, x16, x32
```

Example:

```text
LnkSta: Speed 16GT/s, Width x16
```

---

# 8. MSI Capability

MSI capability ID:

```text
0x05
```

MSI allows a device to signal interrupts using memory writes instead of legacy INTx pins.

Typical MSI fields:

```text
Offset   Field
-----------------------------------------------
0x00     Capability ID
0x01     Next Capability Pointer
0x02     Message Control
0x04     Message Address
0x08     Message Upper Address, if 64-bit capable
0x08/0x0C Message Data
```

Important Message Control fields:

| Field | Meaning |
|---|---|
| MSI Enable | Enables MSI |
| Multiple Message Capable | Number of vectors supported |
| Multiple Message Enable | Number of vectors enabled |
| 64-bit Address Capable | Device supports 64-bit MSI address |
| Per-Vector Masking Capable | Supports per-vector masking |

---

# 9. MSI-X Capability

MSI-X capability ID:

```text
0x11
```

MSI-X is more flexible than MSI and supports more interrupt vectors.

MSI-X capability contains:

```text
Offset   Field
-----------------------------------------------
0x00     Capability ID
0x01     Next Capability Pointer
0x02     Message Control
0x04     Table Offset / Table BAR Indicator
0x08     Pending Bit Array Offset / BAR Indicator
```

The MSI-X table itself is stored in a BAR-mapped memory region, not directly inside configuration space.

Each MSI-X table entry typically contains:

```text
Message Address Lower
Message Address Upper
Message Data
Vector Control
```

Important MSI-X Message Control fields:

| Field | Meaning |
|---|---|
| Table Size | Number of MSI-X vectors minus 1 |
| Function Mask | Masks all MSI-X vectors |
| MSI-X Enable | Enables MSI-X |

---

# 10. PCIe Extended Configuration Space

PCIe extends config space from 256 bytes to 4096 bytes.

The extended area starts at:

```text
0x100
```

Extended capabilities are also organized as a linked list.

Each extended capability starts with a 32-bit header:

```text
Bits 15:0    Extended Capability ID
Bits 19:16   Capability Version
Bits 31:20   Next Capability Offset
```

C-like representation:

```c
struct pcie_ext_cap_header {
    uint32_t cap_id  : 16;
    uint32_t version : 4;
    uint32_t next    : 12;
};
```

The `next` field is an offset in configuration space, usually 4-byte aligned.

A `next` value of `0x000` ends the list.

---

## 10.1 Common PCIe Extended Capability IDs

| ID | Capability |
|---:|---|
| `0x0001` | Advanced Error Reporting, AER |
| `0x0002` | Virtual Channel |
| `0x0003` | Device Serial Number |
| `0x0004` | Power Budgeting |
| `0x000D` | Access Control Services, ACS |
| `0x000E` | Alternative Routing-ID Interpretation, ARI |
| `0x0010` | Single Root I/O Virtualization, SR-IOV |
| `0x0013` | Latency Tolerance Reporting, LTR |
| `0x0015` | Resizable BAR |
| `0x001B` | Precision Time Measurement, PTM |
| `0x001E` | L1 PM Substates |
| `0x0023` | Designated Vendor-Specific Extended Capability, DVSEC |
| `0x0024` | Data Link Feature |
| `0x0025` | Physical Layer 16.0 GT/s |
| `0x0026` | Lane Margining at Receiver |
| `0x0027` | Hierarchy ID |
| `0x002A` | Protocol Multiplexing |
| `0x002B` | Physical Layer 32.0 GT/s |
| `0x002E` | DOE, Data Object Exchange |

---

# 11. Advanced Error Reporting, AER

Extended capability ID:

```text
0x0001
```

AER provides enhanced PCIe error reporting.

It includes registers for:

- Uncorrectable Error Status
- Uncorrectable Error Mask
- Uncorrectable Error Severity
- Correctable Error Status
- Correctable Error Mask
- Advanced Error Capabilities and Control
- Header Log
- Root Error Command
- Root Error Status
- Error Source Identification

Common AER errors include:

Correctable:

- Receiver Error
- Bad TLP
- Bad DLLP
- Replay Timer Timeout
- Advisory Non-Fatal Error

Uncorrectable:

- Data Link Protocol Error
- Surprise Down Error
- Poisoned TLP
- Flow Control Protocol Error
- Completion Timeout
- Completer Abort
- Unexpected Completion
- Unsupported Request
- ECRC Error
- Malformed TLP

---

# 12. SR-IOV Extended Capability

Extended capability ID:

```text
0x0010
```

SR-IOV allows one physical PCIe function to expose multiple virtual functions.

Key terms:

| Term | Meaning |
|---|---|
| PF | Physical Function |
| VF | Virtual Function |

Important SR-IOV fields include:

- SR-IOV Capabilities
- SR-IOV Control
- Initial VFs
- Total VFs
- Number of VFs
- Function Dependency Link
- First VF Offset
- VF Stride
- VF Device ID
- Supported Page Sizes
- System Page Size
- VF BARs
- VF Migration State Array Offset

Operating systems use this capability to enumerate and enable VFs.

---

# 13. Resizable BAR Extended Capability

Extended capability ID:

```text
0x0015
```

Resizable BAR allows software to change supported BAR sizes.

This is often discussed for GPUs.

The capability provides:

- Supported sizes
- Current BAR size
- BAR index

Instead of being fixed at hardware reset, the BAR aperture can be resized by firmware or OS if supported.

---

# 14. Accessing PCIe Configuration Space

## 14.1 Legacy I/O Port Mechanism

On x86, old PCI config access uses ports:

```text
0xCF8
0xCFC
```

This mechanism only accesses the first 256 bytes of configuration space.

---

## 14.2 Enhanced Configuration Access Mechanism, ECAM

PCIe extended config space is commonly accessed through **ECAM**, also called **MMCONFIG**.

The firmware provides the ECAM base address, often through the ACPI `MCFG` table.

Each function gets 4 KiB.

Typical address calculation:

```text
config_address =
    ecam_base
  + ((bus - start_bus) << 20)
  + (device << 15)
  + (function << 12)
  + register_offset
```

Why these shifts?

| Field | Size |
|---|---:|
| Register space per function | 4 KiB = `1 << 12` |
| Functions per device | 8 |
| Device space | `8 * 4 KiB = 32 KiB = 1 << 15` |
| Devices per bus | 32 |
| Bus space | `32 * 8 * 4 KiB = 1 MiB = 1 << 20` |

So:

```text
Bus n consumes 1 MiB of ECAM space.
Device n consumes 32 KiB.
Function n consumes 4 KiB.
```

---

# 15. Example C-Like Structures

## 15.1 Common PCI Header

```c
struct pci_config_common {
    uint16_t vendor_id;
    uint16_t device_id;
    uint16_t command;
    uint16_t status;
    uint8_t  revision_id;
    uint8_t  prog_if;
    uint8_t  subclass;
    uint8_t  base_class;
    uint8_t  cache_line_size;
    uint8_t  latency_timer;
    uint8_t  header_type;
    uint8_t  bist;
};
```

---

## 15.2 Type 0 Endpoint Header

```c
struct pci_type0_header {
    struct pci_config_common common;

    uint32_t bar[6];

    uint32_t cardbus_cis_pointer;

    uint16_t subsystem_vendor_id;
    uint16_t subsystem_id;

    uint32_t expansion_rom_base;

    uint8_t  capabilities_pointer;
    uint8_t  reserved1[3];

    uint32_t reserved2;

    uint8_t  interrupt_line;
    uint8_t  interrupt_pin;
    uint8_t  min_gnt;
    uint8_t  max_lat;
};
```

---

## 15.3 Type 1 Bridge Header

```c
struct pci_type1_header {
    struct pci_config_common common;

    uint32_t bar[2];

    uint8_t  primary_bus_number;
    uint8_t  secondary_bus_number;
    uint8_t  subordinate_bus_number;
    uint8_t  secondary_latency_timer;

    uint8_t  io_base;
    uint8_t  io_limit;
    uint16_t secondary_status;

    uint16_t memory_base;
    uint16_t memory_limit;

    uint16_t prefetchable_memory_base;
    uint16_t prefetchable_memory_limit;

    uint32_t prefetchable_base_upper32;
    uint32_t prefetchable_limit_upper32;

    uint16_t io_base_upper16;
    uint16_t io_limit_upper16;

    uint8_t  capabilities_pointer;
    uint8_t  reserved[3];

    uint32_t expansion_rom_base;

    uint8_t  interrupt_line;
    uint8_t  interrupt_pin;

    uint16_t bridge_control;
};
```

---

## 15.4 PCI Capability Header

```c
struct pci_cap_header {
    uint8_t cap_id;
    uint8_t next_cap;
};
```

---

## 15.5 PCIe Extended Capability Header

```c
struct pcie_ext_cap_header {
    uint32_t header;
};
```

Helper macros:

```c
#define PCI_EXT_CAP_ID(h)      ((h) & 0x0000ffff)
#define PCI_EXT_CAP_VER(h)     (((h) >> 16) & 0xf)
#define PCI_EXT_CAP_NEXT(h)    (((h) >> 20) & 0xfff)
```

---

# 16. Enumeration Summary

A PCIe enumeration algorithm roughly does this:

1. For each bus, device, and function:
   - Read Vendor ID at offset `0x00`
   - If `0xFFFF`, no device exists
2. Read Header Type at offset `0x0E`
3. Determine if it is:
   - Type 0 endpoint
   - Type 1 bridge
4. Read class code and IDs
5. Walk the capability list if Status bit 4 is set
6. For endpoints:
   - Determine BAR sizes
   - Allocate address space
   - Program BARs
   - Enable Memory Space and Bus Mastering
7. For bridges:
   - Assign secondary/subordinate bus numbers
   - Configure bridge memory/I/O windows
8. Walk PCIe extended capabilities starting at `0x100`
9. Configure MSI/MSI-X, power management, AER, SR-IOV, etc.

---

# 17. Key Points

- PCIe configuration space is **4 KiB per function**.
- The first **256 bytes** are compatible with conventional PCI.
- The first **64 bytes** contain a standard header.
- Type 0 headers are used by endpoints.
- Type 1 headers are used by bridges and PCIe ports.
- BARs describe device memory or I/O resources.
- PCI capabilities form an 8-bit linked list below offset `0x100`.
- PCIe extended capabilities form a 12-bit-offset linked list starting at `0x100`.
- MSI/MSI-X, PCIe link information, power management, AER, SR-IOV, and Resizable BAR are all represented through capability structures.
- ECAM/MMCONFIG provides memory-mapped access to the full 4 KiB PCIe configuration space.