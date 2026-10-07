#sriov
In PCIe, a **Virtual Function (VF)** is a lightweight PCIe Function created/provisioned by an **SR-IOV Physical Function (PF)**. It allows one physical PCIe device to expose multiple independently usable PCIe Functions, commonly so different virtual machines can directly access portions of the same hardware.

The simplest model is:
```
Physical PCIe Device
        │
        ├── Physical Function (PF)
        │       │
        │       └── controls SR-IOV
        │
        ├── Virtual Function 0
        ├── Virtual Function 1
        ├── Virtual Function 2
        └── Virtual Function 3
```

> **PF = full-featured Function that manages SR-IOV resources.**  
> **VF = lightweight Function that provides access to a portion of those device resources.**

### Why Virtual Functions exist
Suppose a server has one physical PCIe Ethernet adapter and four virtual machines.

Without SR-IOV, access might involve substantial hypervisor/software involvement:
```
VM1 ──┐
VM2 ──┤
VM3 ──┼──► Hypervisor ──► PF Driver ──► NIC
VM4 ──┘
```

With SR-IOV, the NIC can expose VFs:
```
VM1 ──► VF0 ──┐
VM2 ──► VF1 ──┤
VM3 ──► VF2 ──┼──► Physical NIC Hardware
VM4 ──► VF3 ──┘
```

Each VM can be assigned its own VF, reducing the amount of hypervisor intervention in normal I/O.

### A VF looks like a PCIe Function
A VF is "virtual" in the sense that it represents a partition of a physical device's resources. But from the PCIe/software perspective, it is still a **PCIe Function**. It has its own PCIe identity and Configuration Space.

Conceptually:
```
VF0
┌───────────────────────────┐
│ PCIe Function             │
│                           │
│ Configuration Space       │
│                           │
│ Device identification     │
│ BAR-related resources     │
│ MSI-X capability          │
│ PCIe capabilities         │
│ etc.                      │
└───────────────────────────┘
```

This allows the OS to enumerate and interact with a VF similarly to another PCIe Function, although a VF has a restricted set of configuration capabilities compared with its PF.

### PF creates/enables the VFs
Suppose a NIC initially exposes:
```
NIC
 │
 └── PF0
```

The PF contains the **SR-IOV Extended Capability**. That capability describes things such as the number of VFs supported and the resources associated with them.

Host software can enable VFs:
```
Before:

NIC
 └── PF0


After enabling 4 VFs:

NIC
 ├── PF0
 ├── VF0
 ├── VF1
 ├── VF2
 └── VF3
```

The PF remains responsible for higher-level device/SR-IOV management.

### VFs can generate PCIe transactions
A VF isn't just a software object. The hardware implements the VF such that it can participate in PCIe transactions.

For example, suppose a VF needs to DMA-read host memory:
```
VF0
 │
 │ Memory Read Request
 │
 ▼
PCIe Fabric
 │
 ▼
Root Complex
 │
 ▼
Host Memory
```

The request identifies the VF as the Requester.

The Completion comes back:
```
Host Memory
     │
     │ Completion with Data
     ▼
PCIe Fabric
     │
     ▼
VF0
```

This is important because transactions from different VFs can be distinguished from each other.

### The hardware can separate resources between VFs
Suppose one NIC has:
```
Physical NIC
┌─────────────────────────────────────┐
│                                     │
│              PF0                    │
│                                     │
│   VF0       VF1       VF2       VF3 │
│    │         │         │         │  │
│    ▼         ▼         ▼         ▼  │
│ Queues     Queues    Queues    Queues│
│                                     │
│        Shared NIC hardware          │
│                                     │
└─────────────────────────────────────┘
```

The hardware may give different VFs their own queues, interrupts, addressable resources, and other device-specific resources while sharing the underlying physical hardware.

### PF vs VF

|Physical Function (PF)|Virtual Function (VF)|
|---|---|
|Full-featured PCIe Function|Lightweight PCIe Function|
|Supports SR-IOV management|Does not manage SR-IOV|
|Can enable/configure VFs|Associated with a PF|
|Used by host management|Often assigned to VM/container|
|More configuration/control resources|Reduced configuration/control capabilities|
|Uses physical device resources|Uses allocated/shared physical resources|

A useful way to think about it is:
```
                  Physical Device
                        │
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
            PF0                   PF1
             │
       ┌─────┼─────┐
       ▼     ▼     ▼
      VF0   VF1   VF2
```

A device can potentially contain multiple PFs, and each PF may have its own associated VFs.

### Example with NVMe
For an SR-IOV-capable NVMe controller:
```
                  NVMe SSD
┌──────────────────────────────────────┐
│                                      │
│                 PF                   │
│                                      │
│        ┌─────────┼─────────┐         │
│        ▼         ▼         ▼         │
│       VF0       VF1       VF2        │
│        │         │         │         │
│        ▼         ▼         ▼         │
│     VM #1      VM #2      VM #3      │
│                                      │
│         Shared SSD resources         │
│     Controller / DRAM / NAND         │
│                                      │
└──────────────────────────────────────┘
```

For NVMe specifically, SR-IOV interacts with NVMe's own virtualization/resource-management mechanisms. A VF may get controller resources such as I/O queues and interrupts while the PF retains broader administrative control.

### Most important distinction
Don't interpret **Virtual Function** as "a software-emulated PCIe device." With SR-IOV, the VF is **implemented by the physical PCIe device** and appears to software as a PCIe Function. That is what enables efficient direct I/O.

So the hierarchy to remember is:
```
Physical PCIe Device
       │
       ├── PF
       │    └── Full management/control Function
       │
       ├── VF0
       │    └── Lightweight PCIe Function
       │
       ├── VF1
       │    └── Lightweight PCIe Function
       │
       └── VF2
            └── Lightweight PCIe Function
```

**PF and VF are both PCIe Functions. The key difference is that the PF provides the full SR-IOV management capability, while a VF exposes a limited portion of the physical device's resources for independent use.**

