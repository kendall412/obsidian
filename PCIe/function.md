
Because in PCIe, the **Function is the actual logical entity that performs PCIe transactions and owns Configuration Space**. A physical PCIe device can contain **one or multiple Functions**, so referring only to the physical device would sometimes be ambiguous.

The distinction is:

> **Device = a location/container on a PCIe bus.**  
> **Function = an independently addressable logical PCIe entity at that device location.**

### 1. One physical device can provide multiple Functions
Consider a physical PCIe adapter that contains several capabilities:
```
Physical PCIe Device
┌───────────────────────────────────┐
│                                   │
│ Function 0 → Ethernet Controller  │
│ Function 1 → Storage Controller   │
│ Function 2 → Management Interface │
│                                   │
└───────────────────────────────────┘
```

To software, these can behave like separate PCIe entities.

Each Function has its **own Configuration Space**:
```
Physical Device

Function 0
┌──────────────────────┐
│ 4 KiB Config Space   │
│ Vendor ID            │
│ Device ID            │
│ BARs                  │
│ Capabilities         │
└──────────────────────┘

Function 1
┌──────────────────────┐
│ 4 KiB Config Space   │
│ Vendor ID            │
│ Device ID            │
│ BARs                  │
│ Capabilities         │
└──────────────────────┘

Function 2
┌──────────────────────┐
│ 4 KiB Config Space   │
│ ...                  │
└──────────────────────┘
```

Therefore, saying "send this Configuration Request to the device" isn't sufficiently precise. PCIe needs to know **which Function within that device**.

### 2. This is why PCIe uses BDF
PCIe identifies a Function using:

**Bus : Device : Function**
or **BDF**.

For example:
```
03:00.0
│  │  │
│  │  └── Function 0
│  └───── Device 0
└──────── Bus 3
```

Another Function in the same Device could be:
```
03:00.1
```

Now:
```
03:00.0 ── Function 0
03:00.1 ── Function 1
03:00.2 ── Function 2
```

They share the same:
```
Bus    = 03
Device = 00
```

but are independently addressable because their **Function numbers differ**.

### 3. Configuration Requests target Functions
This ties directly to your earlier questions about Configuration Requests.
Suppose the Root Complex wants to read Configuration Space at `03:00.1`.

Conceptually:
```
Root Complex
     │
     │ Configuration Read
     │
     │ Bus      = 03
     │ Device   = 00
     │ Function = 1
     ▼
PCIe Fabric
     │
     ▼
Physical Device 00
┌──────────────────────────────┐
│                              │
│ Function 0                   │
│                              │
│ Function 1 ◄── target        │
│                              │
│ Function 2                   │
│                              │
└──────────────────────────────┘
```

Function 1's Configuration Space is accessed—not some shared generic "Device Configuration Space."

### 4. Function is therefore the PCIe logical endpoint
This is why the PCIe specification frequently talks about a **Function** rather than simply a "device."

A Function owns or participates in things such as:

```
Function
   │
   ├── Configuration Space
   ├── BARs
   ├── PCIe Capabilities
   ├── Requester ID
   ├── Completer ID
   ├── MSI/MSI-X configuration
   └── Transaction behavior
```

For example, when Function `03:00.0` generates a Memory Read Request, its Requester ID can identify:
```
Requester ID

Bus      = 03
Device   = 00
Function = 0
```

That identifies the **specific Function that originated the request**.

### 5. A simple NVMe SSD usually makes this less obvious
A typical NVMe SSD may expose only one PCIe Function:
```
NVMe SSD
┌──────────────────────────────┐
│                              │
│ Function 0                   │
│ NVMe Controller              │
│                              │
└──────────────────────────────┘

BDF = 03:00.0
```

Because there's only Function 0, it can feel like:
```
device == function
```

But PCIe cannot assume that all devices are single-function.

A multifunction device could expose:
```
Device 00

03:00.0
03:00.1
03:00.2
03:00.3
```

and each is a distinct PCIe Function.

### 6. SR-IOV makes the distinction even more important
SR-IOV introduces:

- [[PF (Physical Function)]]
- [[VF (Virtual Function)]]

For example, a high-performance NIC might expose:
```
Physical NIC
│
├── Physical Function
│
├── Virtual Function
├── Virtual Function
├── Virtual Function
└── Virtual Function
```

These Functions can be independently exposed to software or virtual machines.

Again, the physical hardware device isn't granular enough to identify the entity participating in PCIe transactions.

### The key mental model
Think of the hierarchy like this:

```
Physical PCIe Hardware
        │
        ▼
      Device
        │
        ├── Function 0
        │      └── Configuration Space
        │
        ├── Function 1
        │      └── Configuration Space
        │
        └── Function 2
               └── Configuration Space
```

So when the PCIe Base Specification says **"Function,"** it is usually being deliberately precise:

> **A Function is an independently addressable logical entity within a PCIe device and is the fundamental entity that owns Configuration Space and participates in PCIe transactions.**

That's also why **ECAM allocates 4 KiB per Function**, not merely 4 KiB per physical device.