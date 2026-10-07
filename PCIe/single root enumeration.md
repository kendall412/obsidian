#root_complex
In the PCIe Base Specification, **Single Root enumeration** is the process by which system software discovers and configures the PCIe Functions in a hierarchy controlled by **one Root Complex / Root hierarchy**.

The central idea is:

> **Starting from the Root, software walks the PCIe hierarchy using Configuration Requests, discovers Functions, assigns bus numbers and address resources, and configures the discovered Functions for operation.**

### 1. Start with a single PCIe Root

Consider:
```
                       CPU
                        │
                 ┌──────▼──────┐
                 │ Root Complex│
                 └──────┬──────┘
                        │
                   Root Port
                        │
                        ▼
                  PCIe Switch
                  /         \
                 /           \
                ▼             ▼
             NVMe SSD        NIC
```

There is one Root hierarchy responsible for discovering/configuring everything downstream.

"Single Root" does **not** mean there is only one [[root port]]. A Root Complex may contain multiple Root Ports:
```
                    Root Complex
                 /       |       \
               RP0      RP1      RP2
                │        │        │
               SSD      GPU     Switch
```

This can still be a single-root system.

### 2. Enumeration begins with Configuration Requests
System software probes possible PCIe Functions using **Configuration Read Requests**.

Conceptually, it asks:
```
Does 00:00.0 exist?
Does 00:01.0 exist?
Does 00:02.0 exist?
...
```

For each location, it can read the **Vendor ID**.

If a Function exists:
```
Configuration Read
        │
        ▼
Vendor ID = 0xXXXX
```

If no Function exists at that location, the access does not produce a normal successful device response. This is how software determines what is actually present.

### 3. Software identifies what each Function is

Once a Function is discovered, software reads more of its Configuration Space:
```
Configuration Space

0x00 ── Vendor ID
        Device ID

0x04 ── Command
        Status

0x08 ── Revision ID
        Class Code

0x0C ── Header Type

0x10 ── BAR0
0x14 ── BAR1
        ...

        Capabilities
        Extended Capabilities
```

For example:
```
Function discovered
        │
        ├── Vendor ID
        ├── Device ID
        ├── Class Code
        ├── Header Type
        ├── BARs
        └── Capabilities
```

The **Header Type** is particularly important because it helps software determine whether the Function represents an Endpoint or bridge and whether it is part of a multifunction device.

### 4. Bridges cause enumeration to continue onto another bus
Suppose software discovers a Root Port or PCIe-to-PCIe bridge.

```
Bus 0
 │
 ▼
Root Port
 │
 ▼
????
```

The bridge creates another bus downstream.

Software assigns bus numbers through the bridge's:
```
Primary Bus Number
Secondary Bus Number
Subordinate Bus Number
```

For example:
```
Root Port

Primary     = 0
Secondary   = 1
Subordinate = ...
```

Now:
```
Bus 0
 │
 ▼
Root Port
 │
 ▼
Bus 1
```

Software can begin probing Bus 1.

### 5. Enumeration recursively walks the hierarchy

Suppose Bus 1 contains a switch:
```
Bus 0
 │
 ▼
Root Port
 │
 ▼
Bus 1
 │
 ▼
PCIe Switch
   │
   ├──── Downstream Port
   │          │
   │          ▼
   │        Bus 2
   │          │
   │         SSD
   │
   └──── Downstream Port
              │
              ▼
            Bus 3
              │
             NIC
```

Software continues recursively:
```
Start at Root
     │
     ▼
Scan Bus 0
     │
     ▼
Find bridge
     │
     ▼
Assign Bus 1
     │
     ▼
Scan Bus 1
     │
     ▼
Find more bridges
     │
     ├── assign Bus 2
     │      └── scan Bus 2
     │
     └── assign Bus 3
            └── scan Bus 3
```

This is how the topology is discovered.

### 6. BDFs emerge from this process
After bus numbering, each Function has an identity based on: **Bus : Device : Function**

For example:
```
00:01.0    Root Port

01:00.0    Switch Upstream Port

02:00.0    Switch Downstream Port

03:00.0    NVMe SSD
```

This is the **BDF** you've seen in Configuration Requests.

For example:

```
03:00.0

03 = Bus
00 = Device
0  = Function
```

The Function can now be targeted with Configuration Requests.

### 7. Software determines BAR requirements
Discovery alone isn't enough. Endpoints need address space.

Suppose the NVMe controller has:
```
BAR0
```

Software determines how much address space that BAR requires and allocates an appropriate address range.

Conceptually:
```
NVMe SSD

BAR0 requires:

16 KiB
```

System software may assign:
```
BAR0 = 0x80000000
```

so the NVMe controller's MMIO register space becomes accessible through that host physical address range.

### 8. Bridge windows must also be configured

If the SSD sits behind a bridge:
```
Root Complex
     │
     ▼
Root Port
     │
     ▼
Switch
     │
     ▼
NVMe SSD

BAR0 = 0x80000000
```

the bridges leading to it need address windows that allow transactions targeting that address to be forwarded downstream.

Conceptually:
```
Root Port

Memory Base  = 0x80000000
Memory Limit = 0x80FFFFFF
```

Now an MMIO request to:
```
0x80001000
```

can be routed:
```
CPU
 │
 │ MMIO
 ▼
Root Complex
 │
 ▼
Root Port
 │
 ▼
Switch
 │
 ▼
NVMe SSD
```


### 9. Other capabilities are configured

Enumeration/configuration can also involve capabilities such as:
```
PCIe capabilities
MSI / MSI-X
Power Management
AER
ACS
SR-IOV
Resizable BAR
Device Control
Link Control
```

For an Endpoint, software may eventually enable important Command Register controls such as:
```
Memory Space Enable
Bus Master Enable
```

For example, **Bus Master Enable** allows the Function to originate Memory Requests such as DMA transactions.

### 10. [[ECAM (Enhanced Configuration Access Mechanism)]] is commonly how software performs these accesses

Connecting this to your earlier ECAM question:
Software may calculate:

\[ ECAM + (Bus << 20) + (Device << 15) + (Function << 12) + Offset \]

For an NVMe device at:
```
03:00.0
```

software accesses the corresponding ECAM address.

Then:
```
Software
   │
   │ ECAM access
   ▼
Root Complex
   │
   │ Configuration Request
   ▼
PCIe Fabric
   │
   ▼
Target Function
   │
   │ Completion
   ▼
Root Complex
   │
   ▼
Software
```

ECAM therefore provides the software-facing mechanism for accessing each discovered Function's PCIe Configuration Space.

### Putting the complete enumeration process together

A simplified single-root enumeration flow is:
```
System boots
     │
     ▼
PCIe Root hierarchy available
     │
     ▼
Scan Functions using
Configuration Reads
     │
     ▼
Read Vendor ID
     │
     ├── no Function → continue scanning
     │
     └── Function exists
              │
              ▼
       Read Configuration Space
              │
              ├── Device ID
              ├── Class Code
              ├── Header Type
              ├── BARs
              └── Capabilities
              │
              ▼
       Is it a bridge?
          /          \
        Yes           No
         │             │
         ▼             ▼
 Assign downstream   Configure
 bus number          Endpoint
         │             │
         ▼             │
 Scan downstream      │
 bus                  │
         │             │
         └──────┬──────┘
                ▼
       Allocate resources
                │
                ├── BAR addresses
                ├── bridge windows
                ├── interrupts
                └── other resources
                │
                ▼
       Configure Functions
                │
                ▼
       PCIe hierarchy ready
```

The most important thing is that **enumeration is not just "finding devices."** It is both **discovery and resource configuration**.

For the concepts you've been building up, the relationship is:
```
Single Root
     │
     ▼
Root Complex
     │
     ├── Root Port
     │       │
     │       └── downstream hierarchy
     │
     └── Root Port
             │
             └── downstream hierarchy
                    │
                    ▼
              PCIe Functions
                    │
                    ▼
            Enumeration discovers
            and configures them
```

So, in one sentence:

> **Single-root PCIe enumeration is the host-controlled process of walking one PCIe hierarchy through Configuration Requests, discovering its Functions and bridges, assigning bus numbers and address resources, and configuring those Functions so normal PCIe transactions can begin.**

