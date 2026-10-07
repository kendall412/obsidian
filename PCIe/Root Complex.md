![[pcie_architecture_2.png]]

The root complex in PCI Express (PCIe) is the intermediary between the system’s central processing unit (CPU), memory, and the PCIe switch fabric that includes one or more PCIe or PCI devices. It uses the [[Link Training and Status State Machine (LTSSM)]] to manage connected PCIe devices. The LTSSM detects, polls, configures, recovers, resets, and disables the devices as required during operation.

The PCIe root processes transaction requests received from the CPU and connects to devices on the PCIe bus. A root complex often contains more than one PCIe port; multiple PCIe endpoints, bridges, and switches can connect to the ports on the root complex or be cascaded throughout the system

A master copy of a “Type 1 Configuration Table” is in the root complex and defines the host memory space accessible from each Endpoint device. Each Endpoint device holds the master copy of its own memory space in the host system as a “Type 0 Configuration Table.” Type 1 and Type 0 configuration tables are configured by the host operating system that controls the root complex. A PCIe bridge works as a tiered root complex with its own “Type 0 Configuration Table.” In addition to traditional applications, PCIe is being used as the main system interconnect technology in multi-host and embedded applications. 

## Key functions of the root complex

- Transaction bridging: It translates and bridges transactions between the CPU/memory subsystem and the PCIe bus. 

- Device discovery and configuration: During the boot process, it scans the PCIe bus to detect and identify all connected devices. 

- Resource management: It allocates system resources, such as memory and I/O addresses, to the devices on the bus.

- Communication initiation: As the master of the PCIe hierarchy, it initiates all transactions on behalf of the CPU. 

- Error and power management: It handles error reporting and manages the power states of the connected devices.

---
# PCIe Configuration Request

When the PCIe Base Specification says **only the Root Complex is able/permitted to originate Configuration Requests**, it means:

> **Only the Root Complex can create a new PCIe Configuration Read Request or Configuration Write Request TLP to access a Function's Configuration Space.**

An Endpoint can **receive and complete** a Configuration Request, but it does not initiate one to configure another Endpoint. The PCIe specification explicitly requires the Root Complex to support generation of Configuration Requests as a Requester, while Endpoints support them as Completers. [Intel](https://www.intel.com/content/dam/support/us/en/programmable/support-resources/fpga-wiki/asset03/pci-express-base-r2.1.pdf?utm_source=chatgpt.com)

### What does "originate" mean?

Here, **originate** means **create/initiate the transaction**.

Consider:
```
                 PCIe hierarchy

                    CPU
                     │
                     ▼
              ┌──────────────┐
              │ Root Complex │
              └──────┬───────┘
                     │
                     ▼
                 Root Port
                     │
                     ▼
               ┌──────────┐
               │  Switch  │
               └────┬─────┘
                    │
           ┌────────┴────────┐
           ▼                 ▼
      Endpoint A        Endpoint B
       NVMe SSD            NIC
```

Suppose system software wants to read the **Vendor ID** of the NVMe SSD.

The Root Complex generates a Configuration Read Request:

```
CPU / Software
      │
      │ wants configuration information
      ▼
Root Complex
      │
      │ creates Configuration Read Request TLP
      │
      │ Target BDF = Bus 2, Device 0, Function 0
      ▼
Root Port
      │
      ▼
Switch
      │
      ▼
NVMe Endpoint
```

The NVMe Endpoint receives the request and returns a **Completion with Data**:

```
Root Complex                         NVMe Endpoint
     │                                    │
     │  Configuration Read Request        │
     │───────────────────────────────────►│
     │                                    │
     │       Completion with Data         │
     │◄───────────────────────────────────│
     │                                    │
```

So:

```
Root Complex = Requester
Endpoint     = Completer
```

### What an Endpoint cannot do

Suppose the NVMe SSD is Endpoint A and a NIC is Endpoint B.

Endpoint A **cannot originate a Configuration Request** to read or modify Endpoint B's Configuration Space:

```
NVMe SSD                         NIC
Endpoint A                   Endpoint B

    │                            │
    │ Configuration Read        │
    │ Request                   │
    │────────── X ─────────────►│
    │
    │ NOT ALLOWED
```

For example, the NVMe controller cannot decide:

> "I want to read the NIC's Device Control register."

and generate a Configuration Read Request targeting the NIC.

Configuration Requests originate from the Root Complex side of the hierarchy. This is also why Configuration Requests naturally travel **downstream** and why peer-to-peer Configuration Requests aren't allowed. [O'Reilly Media](https://www.oreilly.com/library/view/pci-express-system/0321156307/0321156307_ch19lev1sec9.html?utm_source=chatgpt.com)

### But Endpoints can originate other PCIe Requests

This does **not** mean an Endpoint cannot originate any PCIe transaction.

An Endpoint can be a Requester for permitted transaction types.

For example, an NVMe controller performs DMA to host memory:

```
                    Host RAM
                       ▲
                       │
                Memory Write
                       │
                Root Complex
                       ▲
                       │
                    Switch
                       ▲
                       │
                   NVMe SSD
```

Here:

```
NVMe Endpoint
     │
     │ Memory Write Request
     ▼
Host Memory
```

The **NVMe Endpoint originates the Memory Write Request**.

Likewise, it can originate Memory Read Requests when appropriate.

So the restriction specifically concerns **Configuration Requests**, not all TLPs.

### Why does PCIe work this way?

Configuration Space controls important properties of a Function, including:

```
Configuration Space
│
├── Vendor ID / Device ID
├── Command Register
├── Status Register
├── BARs
├── PCIe Capability
│    ├── Device Control
│    ├── Device Status
│    ├── Link Control
│    └── Link Status
├── MSI / MSI-X
├── AER
└── Other capabilities
```

During PCIe enumeration, system software needs centralized control over this configuration.

If arbitrary Endpoints could send Configuration Writes to each other:

```
SSD ───► change NIC configuration
NIC ───► change GPU configuration
GPU ───► change SSD configuration
```

the system's configuration could be changed without host software controlling it. Restricting origination to the Root Complex preserves the host-controlled configuration model. [O'Reilly Media](https://www.oreilly.com/library/view/pci-express-system/0321156307/0321156307_ch19lev1sec9.html?utm_source=chatgpt.com)

### One subtle but important point

Don't interpret this as meaning that **the CPU itself constructs a PCIe Configuration TLP**.

Typically:

```
Software
   │
   │ Configuration-space access
   ▼
CPU / Host interface
   │
   ▼
Root Complex
   │
   │ converts/accesses as necessary
   ▼
Configuration Request TLP
   │
   ▼
PCIe fabric
```

The **Root Complex is the PCIe component that actually injects the Configuration Request into the PCIe fabric** on behalf of host software.

So when the spec says:

> **Only the Root Complex may originate Configuration Requests**

the easiest way to remember it is:

**Host configures PCIe devices; PCIe devices don't configure each other.**

This becomes especially important when understanding **Type 0 vs Type 1 Configuration Requests**: the Root Complex starts the request, and switches/bridges determine how that request gets routed down the hierarchy to the target BDF.