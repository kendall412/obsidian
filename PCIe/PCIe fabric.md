#pcie_fabric
In PCIe, the **PCIe fabric** means the interconnected PCIe topology that **transports and routes PCIe transactions between components** such as the Root Complex, switches, bridges, and Endpoints.

Think of the fabric as the **communication network connecting PCIe devices**.

```
                       CPU
                        │
                 ┌──────▼──────┐
                 │ Root Complex│
                 └──────┬──────┘
                        │
                   PCIe Link
                        │
                 ┌──────▼──────┐
                 │ PCIe Switch │
                 └──┬───────┬──┘
                    │       │
              PCIe Link   PCIe Link
                    │       │
              ┌─────▼─┐   ┌─▼─────┐
              │NVMe SSD│   │  NIC  │
              └───────┘   └───────┘

        <------- PCIe Fabric ------->
```

The word **fabric** does not refer to one specific piece of hardware. It refers to the overall interconnected PCIe communication structure.

### What makes up the PCIe fabric?
The fabric includes the **links and routing components** that allow PCIe packets to move through the hierarchy.

For example:
```
Root Complex
     │
     │ PCIe Link
     ▼
Root Port
     │
     │ PCIe Link
     ▼
PCIe Switch
     │
     ├──── PCIe Link ──── NVMe SSD
     │
     └──── PCIe Link ──── NIC
```

Depending on the context, saying a packet travels "through the PCIe fabric" generally means it travels through whatever PCIe links, ports, switches, and bridges lie between the requester and its destination.

### The fabric carries TLPs
At the Transaction Layer, communication occurs using **Transaction Layer Packets (TLPs)**.

For example:
```
Memory Read Request
Memory Write Request
Configuration Read Request
Configuration Write Request
Completion
Completion with Data
Messages
```

Suppose an NVMe SSD DMA-reads host memory:

```
NVMe SSD
(Requester)
     │
     │ Memory Read Request TLP
     ▼
PCIe Switch
     │
     ▼
Root Port
     │
     ▼
Root Complex
     │
     ▼
Host Memory
```

You could describe this as:

> The NVMe controller sends a Memory Read Request through the PCIe fabric toward host memory.

The response travels back:
```
Host side
   │
   │ Completion with Data
   ▼
Root Port
   │
   ▼
PCIe Switch
   │
   ▼
NVMe SSD
```

### The fabric also performs routing
One of the most important functions of the PCIe fabric is getting a TLP to the correct destination.

PCIe has several routing mechanisms, including:
```
Address Routing
ID Routing
Implicit Routing
```

For example, a Memory Request contains an address:
```
Memory Read

Address = 0x8400_1000
```

The PCIe hierarchy uses address ranges configured in ports/bridges to route the request toward the device or host resource responsible for that address.

Configuration Requests are different because they identify a Function using information such as:
```
Bus
Device
Function
```

For example:
```
03:00.0

Bus     = 03
Device  = 00
Function = 0
```

The fabric routes that Configuration Request to the appropriate Function.

### Connecting this to ECAM
This connects directly to your previous question.

Suppose software wants to read PCIe Configuration Space for an NVMe SSD:
```
Software
   │
   │ ECAM memory access
   ▼
Root Complex
   │
   │ generates Configuration
   │ Request TLP
   ▼
════════════════════════════
       PCIe Fabric
════════════════════════════
   │
   ▼
PCIe Switch
   │
   ▼
NVMe Endpoint
```

The **ECAM access happens on the host side**.

The Root Complex then generates the PCIe Configuration Request. That Configuration Request travels **through the PCIe fabric** to the target device.

The Endpoint's Completion travels back through the fabric:
```
CPU
 │
 ▼
Root Complex
 │
 │ Configuration Request
 ▼
═══════════════════════════
        PCIe Fabric
═══════════════════════════
 │
 ▼
NVMe SSD
 │
 │ Completion
 ▼
═══════════════════════════
        PCIe Fabric
═══════════════════════════
 │
 ▼
Root Complex
 │
 ▼
CPU
```

### Fabric vs link
These terms are easy to confuse.

A **PCIe link** is a point-to-point connection between exactly two PCIe components:
```
Root Port ═════════ NVMe SSD
              ↑
          PCIe Link
```

A **PCIe fabric** refers to the larger interconnected system:
```
             Root Complex
                  │
                 Link
                  │
                Switch
              /    |    \
           Link   Link   Link
            /      |      \
          SSD     GPU      NIC

          < PCIe Fabric >
```

So:

> **Link = one point-to-point connection.**

> **Fabric = the interconnected PCIe communication topology through which transactions are routed.**

For your PCIe/NVMe studies, when the Base Spec says something like **"a TLP is routed through the fabric,"** mentally picture a TLP moving from a **Requester → links/switches/ports → Completer**.

