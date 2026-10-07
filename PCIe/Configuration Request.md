#pcie_configuration_space #configuration_request
In the PCIe Base Specification, a **Configuration Request** is a [[PCIe TLP (PCIe Transaction Layer Packet)]] used to **read or write a PCIe Function's Configuration Space**. Configuration transactions are specifically intended for device/function configuration and setup. [Studylib](https://studylib.net/doc/28952562/pci-sig.-pci-express-base-specification--revision-6.0--ve...?utm_source=chatgpt.com) Only the [[Root Complex]] is allowed to make Configuration Request.

For example, when the host wants to discover an NVMe SSD's Vendor ID, Device ID, BARs, or PCIe capabilities, it accesses the device's Configuration Space using Configuration Requests.

### The four Configuration Request TLPs
PCIe defines four Configuration Request types: [Studylib](https://studylib.net/doc/28952562/pci-sig.-pci-express-base-specification--revision-6.0--ve...?utm_source=chatgpt.com)

|TLP|Meaning|
|---|---|
|`CfgRd0`|Configuration Read Type 0|
|`CfgWr0`|Configuration Write Type 0|
|`CfgRd1`|Configuration Read Type 1|
|`CfgWr1`|Configuration Write Type 1|

So there are two operations:
```
Configuration Read
Configuration Write
```

and two routing types:
```
Type 0
Type 1
```


### 1. What is being read or written?
Every PCIe Function has **Configuration Space**.

Conceptually:
```
PCIe Function Configuration Space

Offset
0x000 ┌─────────────────────────┐
      │ Vendor ID               │
0x002 │ Device ID               │
      ├─────────────────────────┤
0x004 │ Command                 │
0x006 │ Status                  │
      ├─────────────────────────┤
0x010 │ BAR0                    │
0x014 │ BAR1                    │
      ├─────────────────────────┤
      │ ...                     │
      ├─────────────────────────┤
      │ PCIe Capabilities       │
      │ Extended Capabilities   │
      └─────────────────────────┘
0xFFF
```

PCIe supports up to **4 KiB of Configuration Space per Function**.
A Configuration Request tells the target Function:

> "Read this location in your Configuration Space."

or:

> "Write this value into this location in your Configuration Space."

### 2. Configuration Read example
Suppose the Root Complex wants to determine the Vendor ID and Device ID of an NVMe SSD.

It sends something conceptually like:
```
Root Complex

Configuration Read Request
    │
    │ Target:
    │ Bus      = 03
    │ Device   = 00
    │ Function = 0
    │ Register = 0x000
    │
    ▼
NVMe SSD
```

The Endpoint receives the request and reads its Configuration Space:
```
Offset 0x000

31                    16 15                     0
┌───────────────────────┬────────────────────────┐
│      Device ID        │       Vendor ID        │
└───────────────────────┴────────────────────────┘
```

It then returns a **Completion with Data (CplD)**. The PCIe specification identifies `CplD` as the Completion type used for successful Configuration Reads. [Studylib](https://studylib.net/doc/28952562/pci-sig.-pci-express-base-specification--revision-6.0--ve...?utm_source=chatgpt.com)

So:
```
Root Complex                         NVMe Endpoint
     │                                    │
     │ Configuration Read Request         │
     │───────────────────────────────────►│
     │                                    │
     │ Completion with Data (CplD)        │
     │◄───────────────────────────────────│
     │                                    │
```


### 3. Configuration Write example
Suppose software wants to modify the PCIe **Command Register**.

The Root Complex sends:
```
Configuration Write Request

Target BDF = 03:00.0
Register   = 0x004
Data       = xxxx
```

The Endpoint receives the request:
```
Root Complex                         NVMe Endpoint
     │                                    │
     │ Configuration Write Request        │
     │───────────────────────────────────►│
     │                                    │
     │          Completion (Cpl)           │
     │◄───────────────────────────────────│
```

Unlike a successful Configuration Read, the successful Configuration Write completion doesn't need to return the written data. The specification defines `Cpl` without data for Configuration Write completions. [Studylib](https://studylib.net/doc/28952562/pci-sig.-pci-express-base-specification--revision-6.0--ve...?utm_source=chatgpt.com)

### Type 0 vs Type 1
This is one of the most important parts of understanding Configuration Requests.

- **Type 0** is used when the request has reached the bus containing the target Function.
- **Type 1** is used for routing the request through PCIe bridges/switches toward another bus.

Consider:
```
                    Root Complex
                         │
                         │
                    Root Port
                         │
                         ▼
                     Switch
                    /      \
                   /        \
                  ▼          ▼
               NVMe SSD     NIC
```

Suppose the NVMe SSD is:
```
Bus      = 03
Device   = 00
Function = 0
```

or:
```
BDF = 03:00.0
```

The configuration request starts from the Root Complex and is routed down the PCIe hierarchy.

Conceptually:
```
Root Complex
     │
     │ Configuration Request
     ▼
Root Port
     │
     │ Type 1 while routing
     ▼
PCIe Bridge/Switch
     │
     │ Type 0 when targeting
     │ device on that bus
     ▼
NVMe Endpoint
```

So a useful mental model is:
```
Type 1
    │
    │ "Route this toward another bus."
    ▼
Bridge
    │
    │ Convert when target bus reached
    ▼
Type 0
    │
    │ "The target Function is on this bus."
    ▼
Endpoint
```


### Configuration Requests use ID routing
Memory Requests usually identify their target using an **address**:
```
Memory Request

Address = 0x8000_1000
```

Configuration Requests instead identify the target Function using:
```
Bus
Device
Function
```

or **BDF**.

For example:
```
03:00.0

03 = Bus
00 = Device
0  = Function
```

The Configuration Request header therefore contains information identifying the target Function and the register being accessed.

Conceptually:
```
Configuration Request TLP

┌───────────────────────────────────────┐
│ TLP Type: CfgRd0 / CfgWr0 / etc.     │
├───────────────────────────────────────┤
│ Requester ID                          │
├───────────────────────────────────────┤
│ Tag                                   │
├───────────────────────────────────────┤
│ Bus Number                            │
│ Device Number                         │
│ Function Number                       │
├───────────────────────────────────────┤
│ Register Number                       │
├───────────────────────────────────────┤
│ Data (Configuration Write only)       │
└───────────────────────────────────────┘
```

The actual header is a **3-DWORD header**; Configuration Reads have no data payload, while Configuration Writes carry data. The Fmt/Type encodings distinguish `CfgRd0`, `CfgWr0`, `CfgRd1`, and `CfgWr1`. [Studylib](https://studylib.net/doc/28952562/pci-sig.-pci-express-base-specification--revision-6.0--ve...?utm_source=chatgpt.com)

### Why Configuration Requests exist
During **PCIe enumeration**, the host needs to discover and configure every PCIe Function.

A simplified sequence is:
```
System boots
    │
    ▼
Root Complex searches PCIe hierarchy
    │
    ▼
Configuration Read
    │
    ├── Vendor ID
    ├── Device ID
    ├── Class Code
    ├── Header Type
    └── Capabilities
    │
    ▼
Device discovered
    │
    ▼
Configuration Writes
    │
    ├── Configure BARs
    ├── Enable Memory Space
    ├── Enable Bus Mastering
    ├── Configure MSI/MSI-X
    └── Configure PCIe capabilities
    │
    ▼
Device ready for normal operation
```

That ties directly into your previous question about **"only the Root Complex can originate Configuration Requests."** The Root Complex initiates these transactions because the host owns PCIe discovery and configuration. Endpoints such as an NVMe SSD respond as **Completers**; they don't initiate Configuration Requests to configure another Endpoint.

For an NVMe SSD, it's also important not to confuse **PCIe Configuration Space** with the **NVMe Controller Registers** (`CAP`, `CC`, `CSTS`, `AQA`, etc.). Configuration Requests access PCIe Configuration Space; after BARs are configured, the host normally accesses NVMe controller registers through **Memory Read/Write transactions (MMIO)**, not Configuration Requests. [Studylib](https://studylib.net/doc/28952562/pci-sig.-pci-express-base-specification--revision-6.0--ve...?utm_source=chatgpt.com)

# Generating Configuration Transactions

Configuration space can be accessed using either of two mechanism
- The legacy PCI configuration mechanism, using IO-indirect accesses.
- The [[ECAM (Enhanced Configuration Access Mechanism)]], using memory-mapped accesses.

