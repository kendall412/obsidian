#root_port
A **Root Port** in PCIe is a **Port on the Root Complex that connects the Root Complex to a downstream PCIe hierarchy**.

In simple terms:

> **The Root Complex connects the CPU/memory system to PCIe, and a Root Port is one of the PCIe connection points through which PCIe devices are attached.**

### 1. Basic topology
A system might look like this:
```
                     CPU
                      │
               ┌──────▼──────┐
               │ Root Complex│
               │             │
               │ RP0 RP1 RP2 │
               └─┬───┬───┬───┘
                 │   │   │
                 │   │   └────────────┐
                 │   │                │
                 ▼   ▼                ▼
               NVMe GPU          PCIe Switch
                                    │
                              ┌─────┴─────┐
                              ▼           ▼
                            NVMe         NIC
```

Here the Root Complex contains three Root Ports:
```
Root Port 0 → NVMe SSD
Root Port 1 → GPU
Root Port 2 → PCIe Switch
```

Each Root Port starts a **branch of the PCIe hierarchy**.

### 2. A Root Port is a PCIe bridge
Architecturally, a Root Port behaves as a **PCI-PCI Bridge structure** between the Root Complex and its downstream PCIe hierarchy.

For example:
```
Host / Root Complex side
          │
          ▼
    ┌─────────────┐
    │  Root Port  │
    │             │
    │ PCIe Bridge │
    └──────┬──────┘
           │
           │ PCIe Link
           ▼
        NVMe SSD
```

Because it acts like a bridge, the Root Port has Configuration Space containing bridge-related information such as:

```
Primary Bus Number
Secondary Bus Number
Subordinate Bus Number

Memory Base
Memory Limit

I/O Base
I/O Limit

PCIe Capability
AER capability
etc.
```

These values are important for routing transactions into the hierarchy below that Root Port.

### 3. Root Port vs Root Complex
These are not the same thing.

```
             Root Complex
┌──────────────────────────────────┐
│                                  │
│ Host/PCIe interface              │
│                                  │
│  ┌─────────┐   ┌─────────┐      │
│  │Root Port│   │Root Port│      │
│  │   #0    │   │   #1    │      │
│  └────┬────┘   └────┬────┘      │
│       │             │            │
└───────┼─────────────┼────────────┘
        │             │
        ▼             ▼
      NVMe           GPU
```

The **Root Complex** is the broader host-side PCIe subsystem.
A **Root Port** is a specific downstream-facing PCIe Port within that Root Complex.

Therefore:
```
One Root Complex

        │
        ├── Root Port 0
        ├── Root Port 1
        ├── Root Port 2
        └── Root Port 3
```

is completely normal.

### 4. Root Port vs PCIe link
A Root Port is also not the link itself.

```
┌─────────────┐                 ┌──────────────┐
│  Root Port  │═════════════════│   NVMe SSD   │
└─────────────┘                 └──────────────┘
                     ↑
                  PCIe Link
```

The components are:
```
Root Port
    │
    │ transmitter/receiver
    ▼
PCIe Link
    │
    ▼
Endpoint Port
```

The link could be, for example:
```
PCIe Gen5 x4
```

The Root Port is one end of that link.

### 5. Root Ports participate in enumeration
Suppose you have:
```
Root Complex
     │
     ▼
Root Port
     │
     ▼
PCIe Switch
     │
     ▼
NVMe SSD
```

During enumeration, software discovers the Root Port and configures its bus-number registers.

For example:
```
Root Port

Primary Bus     = 0
Secondary Bus   = 1
Subordinate Bus = 5
```

This tells the hierarchy that buses `1–5` exist downstream of this Root Port.

Conceptually:

```
Bus 0

Root Complex
     │
     ▼
Root Port
Primary     = 0
Secondary   = 1
Subordinate = 5
     │
     ▼
Bus 1
     │
     ▼
PCIe Switch
     │
     ├── Bus 2
     ├── Bus 3
     └── Bus 4
```

This is one way Configuration Requests get routed toward downstream Functions.

### 6. Root Ports also route memory transactions
Suppose an NVMe SSD has a BAR assigned to:
```
BAR0 = 0x80000000
```

Software performs MMIO to:
```
0x80001000
```

The host-side PCIe infrastructure determines which downstream hierarchy owns that address.

Conceptually:
```
CPU
 │
 │ MMIO
 │ 0x80001000
 ▼
Root Complex
 │
 ▼
Root Port
 │
 │ Memory Request
 ▼
PCIe Link
 │
 ▼
NVMe SSD
```

The Root Port is therefore part of the path between the host and downstream PCIe devices.

### 7. Root Ports also handle PCIe-specific features
Because a Root Port is itself a PCIe Function/bridge structure, it can implement PCIe capabilities such as:
```
Link Control / Status
Device Control / Status
Advanced Error Reporting (AER)
Power Management
ASPM
Link speed control
Link width reporting
Hot-plug support
ACS
DPC
```

depending on what the platform supports.

For example, if you inspect Linux:

```
lspci -t
```

you might conceptually see:

```
CPU / Root Complex
       │
       └── 00:01.0  Root Port
                │
                └── 01:00.0 NVMe Controller
```

`00:01.0` is the Root Port Function.

`01:00.0` is the NVMe Endpoint Function.

### 8. Root Port vs Switch Port
They perform similar bridging/routing roles but exist at different locations.

```
                   Root Complex
                        │
                   Root Port
                        │
                        ▼
                  ┌──────────┐
                  │  Switch  │
                  └────┬─────┘
                       │
              ┌────────┴────────┐
              ▼                 ▼
        Downstream Port   Downstream Port
              │                 │
              ▼                 ▼
            NVMe               NIC
```

The Root Port is the bridge from the **Root Complex into a PCIe hierarchy**. A switch's ports exist within the PCIe fabric farther downstream.

### The key mental model
Think of the relationship as:
```
CPU / Memory
     │
     ▼
Root Complex
     │
     ├── Root Port 0 ── PCIe Link ── NVMe
     │
     ├── Root Port 1 ── PCIe Link ── GPU
     │
     └── Root Port 2 ── PCIe Link ── Switch
                                      │
                                      ├── NIC
                                      └── NVMe
```

So the three terms you've been asking about fit together as:

- **Root Complex** = the host-side PCIe subsystem.
- **Root Port** = a PCIe port/bridge within the Root Complex that begins a downstream PCIe hierarchy.
- [[PCIe fabric]] = the interconnected links, ports, switches, and other routing infrastructure through which PCIe transactions travel.

And importantly, **having multiple Root Ports does not mean you have multiple Root Complexes**. One Root Complex commonly contains many Root Ports.