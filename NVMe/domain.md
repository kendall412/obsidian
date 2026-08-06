# Domain

In NVMe, <u>a domain is a logical partition of an NVMe subsystem that groups together controllers and namespaces so they can operate somewhat independently from other</u> groups within the same subsystem.

The domain concept was introduced in newer NVMe specifications (NVMe 2.x) <u>to support large enterprise SSDs, multi-controller systems, and storage arrays</u>.

### Why domains exist
Domains allow a single NVMe subsystem to be divided into independent regions that can:
- Isolate resources between different hosts or applications.
- Improve scalability in multi-controller SSDs.
- Allow maintenance or resets on one domain without affecting another.
- Support high-availability and enterprise storage architectures.

### Relationship to other NVMe objects
```
NVMe Subsystem
│
├── Domain 0
│   ├── Controller A
│   ├── Controller B
│   └── Namespaces assigned to Domain 0
│
└── Domain 1
    ├── Controller C
    ├── Controller D
    └── Namespaces assigned to Domain 1
```

Each domain contains:

- One or more NVMe controllers
- One or more namespaces
- Resources allocated specifically to that domain

### Domains vs. other NVMe concepts
|Concept|Purpose|
|---|---|
|Subsystem|The entire NVMe storage device or storage system|
|Domain|A logical partition within a subsystem|
|Controller|Interface through which a host communicates with the subsystem|
|Namespace|A logical block storage device presented to the host|

#### Example
Imagine a storage array serving two customers:

- Domain 0
    - Controllers 0 and 1
    - Namespaces 1–50
    - Customer A
- Domain 1
    - Controllers 2 and 3
    - Namespaces 51–100
    - Customer B

Although both customers use the same physical NVMe subsystem, their resources are isolated by domains.

### Domains vs. Endurance Groups and NVM Sets
These are different concepts:

- Domain → partitions controllers and namespaces for management and isolation.
- Endurance Group → groups flash media that share endurance characteristics and wear information.
- NVM Set → a subset of an Endurance Group with its own allocation of storage resources.

```
Subsystem
├── Domain 0
│   ├── Controllers
│   └── Namespaces
│
└── Endurance Group
     ├── NVM Set 1
     ├── NVM Set 2
     └── Physical flash
```

### Do typical consumer SSDs use domains?

Usually no. Most consumer NVMe SSDs expose:
- One subsystem
- One controller
- No domain functionality

Domains are primarily found in:

- Enterprise NVMe SSDs
- Multi-controller NVMe devices
- NVMe storage appliances
- High-availability storage systems

For someone learning NVMe firmware, domains are considered an advanced enterprise feature. Understanding controllers, namespaces, queues, and the PCIe interface is much more fundamental before diving into domains.

