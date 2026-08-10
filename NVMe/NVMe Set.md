
  > An NVM Set (Non-Volatile Memory Set) is an NVMe concept introduced for enterprise SSDs that **groups a portion of the physical flash media within an Endurance Group. Each NVM Set has its own storage resources and is intended to improve performance isolation, predictability, and media management**.

### Hierarchy
An NVM Set fits into the NVMe hierarchy like this:
```
NVMe Subsystem
│
├── Controller(s)
│
├── Namespace(s)
│     │
│     └── Associated with an NVM Set
│
└── Endurance Group(s)
      │
      ├── NVM Set 1
      ├── NVM Set 2
      └── NVM Set 3
```

### Why NVM Sets exist

Without NVM Sets, all namespaces may compete for the same pool of flash resources. This can cause:
- Unpredictable latency
- Write interference between workloads
- Uneven wear of NAND flash

By dividing flash into separate NVM Sets, the SSD can better isolate workloads.

#### For example:
- NVM Set 1
    - Used for a latency-sensitive database
    - Uses a dedicated portion of NAND
- NVM Set 2
    - Used for backups
    - Uses a different portion of NAND

Heavy backup writes are less likely to impact the database workload.

### Relationship to Namespaces

Each namespace belongs to one NVM Set.

#### For example:
```
Endurance Group
│
├── NVM Set 1
│     ├── Namespace 1
│     └── Namespace 2
│
└── NVM Set 2
      ├── Namespace 3
      └── Namespace 4
```

When a host writes to Namespace 1, the data is stored only within the flash resources allocated to NVM Set 1.

### NVM Set vs. Endurance Group

|Feature|Endurance Group|NVM Set|
|---|---|---|
|Scope|Larger grouping of flash resources|Subdivision within an Endurance Group|
|Purpose|Manage endurance and wear reporting|Isolate storage resources and workloads|
|Contains|One or more NVM Sets|Namespaces|
|Granularity|Coarse|Fine|

### NVM Set vs. Domain

These concepts serve different purposes:

|Domain|NVM Set|
|---|---|
|Organizes controllers and namespaces|Organizes physical storage resources|
|Used for subsystem partitioning|Used for flash resource allocation|
|Management and access isolation|Performance and media isolation|

### Do consumer SSDs use NVM Sets?
Generally no. Most consumer NVMe SSDs have:

- One Endurance Group
- One NVM Set
- One controller

NVM Sets are mainly used in:

- Enterprise NVMe SSDs
- Cloud storage infrastructure
- High-performance data centers
- Multi-tenant storage systems

For someone learning NVMe firmware, NVM Sets are an advanced feature. It's more important to first understand namespaces, submission/completion queues, the Flash Translation Layer (FTL), and wear leveling, then move on to Endurance Groups and NVM Sets.