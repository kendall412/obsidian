
In NVMe, a namespace is the logical storage object that a host reads from and writes to. It is roughly comparable to a SCSI LUN or a block device exposed by a storage controller, but NVMe defines it with NVMe-specific attributes such as namespace identifiers, LBA formats, protection information, multipath sharing, reservations, and namespace management.

A single NVMe subsystem may expose one namespace or many namespaces. Each namespace is addressed by a Namespace Identifier, or NSID, and I/O commands include an NSID field so that the controller knows which namespace the command targets.

## 1. What a namespace represents<br>
**A namespace is a quantity of non-volatile storage that is formatted into logical blocks and made accessible to one or more NVMe controllers**.

Conceptually:

```
NVMe Subsystem
 ├── Controller 1
 ├── Controller 2
 ├── Namespace 1
 ├── Namespace 2
 └── Namespace 3
```
A host sends commands to a controller, and the command specifies the namespace to access.

For example, a Read command contains:

```
Opcode = Read
NSID   = 1
SLBA   = starting logical block address
NLB    = number of logical blocks
```
The controller interprets that command as a read from namespace 1.

## 2. Namespace Identifier, NSID<br>
Each namespace is identified by a 32-bit Namespace Identifier, abbreviated NSID.

Important properties:

|Item|Descritption|
|---|---|
|NSID|32-bit identifier used in commands|
|Scope|Unique within an NVMe subsytem for active namespaces|
|NSID 0|Not used as a normal namepsace identifier|
|NSID `FFFFFFFFh`|Special broadcast value used by some commands to mean "all namespaces"|
|Lifetime|Mya persist, but host software should use namespace identification descriptors for stable identity|

```
Namespace 1 -> NSID = 1
Namespace 2 -> NSID = 2
Namespace 3 -> NSID = 3
```

The NSID is the main handle used by I/O commands.

## 3. Namespace versus controller<br>
- A namespace and a controller are separate concepts.
- A controller is the NVMe interface that accepts commands.
- A namespace is the logical storage region that commands operate on.
- A namespace may be attached to:
    1. One controller only.
    2. Multiple controllers in the same NVMe subsystem.
    3. No controller, if it is allocated but detached.

This is especially important in enterprise NVMe and NVMe over Fabrics environments.

Example:
```
NVMe Subsystem
 ├── Controller A
 │    ├── Namespace 1
 │    └── Namespace 2
 │
 ├── Controller B
 │    ├── Namespace 1
 │    └── Namespace 3
 │
 ├── Namespace 1  shared
 ├── Namespace 2  private to Controller A
 └── Namespace 3  private to Controller B
```

## 4. Private and shared namespaces<br>
NVMe supports both private and shared namespaces.

#### Private namespace
A private namespace is attached to only one controller.

Example:
```
Controller 1 -> Namespace 1
Controller 2 -> Namespace 2
```

#### Shared namespace
A shared namespace is attached to multiple controllers.

Example:
```
Controller 1 -> Namespace 1
Controller 2 -> Namespace 1
```

This is used for multipath I/O, high availability, virtualization, and clustered storage.

The Identify Namespace data structure contains the NMIC, or Namespace Multi-path I/O and Namespace Sharing Capabilities field. This indicates whether the namespace may be attached to more than one controller.

## 5. Active, allocated, and attached namespaces<br>
NVMe distinguishes between namespace existence and namespace accessibility.

#### Allocated namespace<br>
An allocated namespace has been created inside the NVMe subsystem. It consumes namespace management resources and may represent reserved storage capacity. However, **an allocated namespace is not necessarily usable by a controller**.

#### Attached namespace<br>
**A namespace is attached to a controller when that controller is allowed to access it**. Only attached namespaces are visible for normal I/O through that controller.

#### Active namespace<br>
The term active is used for namespaces that are valid and accessible in the relevant context. **From the point of view of a controller, an active namespace is generally one that is attached to that controller**.

A namespace can therefore be:

|State|Meaning|
|---|---|
|Not allocated|Namespace does not exist|
|Allocated but detached|Exists in the subsystem but not accessible through a given controller|
|Attached/active|Accessible through a controller for I/O|

## 6. Namespace creation and deletion<br>
Namespace creation and deletion are optional capabilities. They are controlled by the Namespace Management feature set. If supported, the host can dynamically create and delete namespaces.

Relevant admin commands include:

|Command|Purpose|
|---|---|
|Identify|Discover namespace information|
|Namespace Mgmt|Create or delete namespaces|
|Namespace Attachment|Attach or detach namespaces to controllers|
|Format NVM|Format a namespace|
|Get Log page|Retrieve namespace-related logs|

Namespace management is typically used in enterprise SSDs, composable infrastructure, virtualized environments, and NVMe-oF storage arrays. Many consumer NVMe SSDs expose a single fixed namespace and do not support namespace management.

## 7. Namespace attachment<br>
After a namespace is created, it may need to be attached to one or more controllers. The Namespace Attachment command supports operations such as:

    - Attach namespace to controller list.
    - Detach namespace from controller list.

This allows one namespace to be accessed by multiple controllers, or to be isolated to a specific controller.

Example:
```
Namespace 5 created
Namespace 5 attached to Controller 1
Namespace 5 attached to Controller 2
```

Now both controllers may expose NSID 5.

## 8. Namespace size, capacity, and utilization<br>
A namespace has several important size-related fields in the Identify Namespace data structure.

#### NSZE — Namespace Size 
#nsze
`NSZE` indicates the total size of the namespace in logical blocks.

If:

```
NSZE = 1,000,000
LBA size = 4096 bytes
```

Then the namespace addressable size is:

```
1,000,000 × 4096 = 4,096,000,000 bytes
```

#### NCAP — Namespace Capacity
#ncap
`NCAP` indicates the maximum number of logical blocks that may be allocated in the namespace.

- For fully provisioned namespaces, NCAP is usually equal to NSZE.
- For thin-provisioned namespaces, NCAP may be smaller than NSZE.

#### NUSE — Namespace Utilization
#nuse
`NUSE` indicates the number of logical blocks currently allocated or in use.

This is most relevant for thin-provisioned namespaces.

Example:
```
NSZE = 1,000,000 LBAs
NCAP =   800,000 LBAs
NUSE =   250,000 LBAs
```

Meaning:

- The namespace address space contains 1,000,000 logical blocks.
- At most 800,000 logical blocks may be physically allocated.
- 250,000 logical blocks are currently allocated or used.

## 9. Logical blocks and LBA formats<br>
Namespaces are formatted into logical blocks. A namespace may support several LBA formats, but only one is active at a time.

An LBA format defines:

|Field|Meaning|
|---|---|
|Logical block data size|Usually 512 bytes, 4096 bytes, etc|
|Metadata size|Optional metadata bytes per logical block|
|Relative performance|Hint about relative performance of the format|

Common logical block sizes:

```
512 bytes
4096 bytes
8192 bytes
```

The active format is identified through the Formatted LBA Size, or FLBAS, field.

Example LBA format table:

|LBA Format|Data Size|Metadata size|
|:-:|:-:|:-:|
|0|512 B|0 B|
|1|4096 B|0 B|
|2|4096 B|8 B|
|3|4096 B|16 B|

If the namespace is formatted with LBA Format 1, each logical block is 4096 bytes with no metadata.

## 10. Metadata<br>
NVMe namespaces may support metadata associated with each logical block.

Metadata can be used for:

- End-to-end protection information.
- Application tags.
- Reference tags.
- Storage stack integrity checking.
- Vendor-specific purposes.

Metadata may be transferred in one of two ways:

#### Separate metadata buffer<br>
The command provides a separate metadata pointer.
```
Data buffer     -> user data
Metadata buffer -> metadata
```

Extended LBA format
Metadata is appended to each logical block.

Example:
```
4096 bytes data + 8 bytes metadata = 4104 bytes transferred per block
```
The namespace reports its metadata capabilities in the Identify Namespace data structure.

## 11. Protection Information<br>
NVMe supports end-to-end data protection using Protection Information, often abbreviated PI.

Protection Information can include:

|Field|Purpose|
|---|---|
|Guard|CRC or checksum-like protection|
|Application Tag|Host/application-defined tag|
|Reference Tag|Usually related to logical block address|

NVMe supports multiple PI types, commonly known as Type 1, Type 2, and Type 3 protection information.

The namespace reports protection capabilities through fields such as:

|Field|Meaning|
|---|---|
|DPC|Data Protection Capabilities|
|DPS|Data Protection Settings|

- `DPC` tells the host what protection formats are supported.
- `DPS` tells the host what protection format is currently enabled.

12. Namespace formatting<br>
The Format NVM command changes the format of a namespace.

A format operation may select:

- LBA format.
- Metadata settings.
- Protection Information settings.
- Secure erase behavior, depending on command parameters and controller support.

Formatting may destroy existing user data.<br>
Some controllers support formatting a single namespace. Others may format all namespaces together, depending on the controller’s capabilities. This behavior is reported in the Identify Controller data structure, including fields such as FNA, the Format NVM Attributes field.

13. Namespace identification<br>
NSID is useful for command routing, but it is not always enough for persistent identity. NVMe therefore provides additional namespace identifiers.

Common namespace identifiers include:

|Identifier|Description|
|---|---|
|EUI-64|IEEE Extended Unique Identifier|
|NGUID|Namespace Globally Unique Identifier|
|UUID|Universally Unique Identifier|
|Namespace Identification Descriptor List|Extensible list of identifiers|

These allow host software to recognize the same namespace across resets, re-enumeration, path changes, or controller changes.

For example, in a multipath system, the same namespace might appear through two controllers. The host can compare namespace identifiers to determine that both paths reach the same storage object.

14. Namespace Identification Descriptor List<br>
NVMe defines an Identify operation that returns a namespace identification descriptor list.

This list may contain descriptors such as:

- EUI-64
- NGUID
- UUID
- Command Set Identifier-related information

The descriptor format is extensible. Each descriptor has a type, length, and value. This is important because newer NVMe revisions and command sets can add new identifier types without changing the basic Identify command model.

15. Namespace and command sets<br>
Starting with NVMe 2.x, the specification architecture separates the base specification from command set specifications. The base specification defines common namespace concepts, controller behavior, queues, admin commands, discovery, and management models.

Command set specifications define command-set-specific namespace behavior. Examples include:

|Commmand Set|Namespace Type|
|---|---|
|NVM Command Set|Traditional block namespaces|
|Zoned Namespace Command Set|Zone block namespaces|
|Key Value Command Set|Key-value namespaces, where supported|

**A namespace is associated with an I/O command set**. The host must use the appropriate command set for that namespace.

For example:

- An NVM namespace accepts normal Read, Write, Flush, Dataset Management, and Write Zeroes commands.
- A Zoned Namespace accepts zone-aware commands and obeys zone write rules.

16. Namespace and multipath access<br>
NVMe supports multiple paths to the same namespace.

This is common in:

- NVMe over Fabrics.
- Dual-port NVMe SSDs.
- Storage arrays.
- High-availability systems.

When a namespace is accessible through multiple controllers, the host may see multiple controller paths to the same namespace.

NVMe provides features to help manage this, including:

|Feature|Purpose|
|---|---|
|Namespace identifiers|Determine whether paths refere to the same namespaces|
|ANA|Asymmetric Namespace Access paht state reporting|
|Reserverations|Coordinate shared access|
|Namespace sharing capability|Indicates whether namespace may be shared|

17. ANA — Asymmetric Namespace Access
 #ana
Asymmetric Namespace Access, or ANA, is used when a namespace can be accessed through multiple controllers or paths with different access characteristics.

A path may be:

|ANA State|Meaning|
|---|---|
|Optimized|Best path for I/O|
|Non-optimized|Usable gbut not preferred|
|Inaccessible|Not currently usable|
|Persistent Loss|Path is permanently unavailable|
|Change|ANA state transition in progress|

A namespace is associated with an ANA group. The Identify Namespace data may include an ANA Group Identifier.

The host can read ANA log pages to determine which controller paths are optimal for a namespace.

18. Reservations and shared namespaces
#reservation 
When a namespace is shared by multiple hosts or controllers, coordination is necessary.

NVMe reservations provide a way to control access to a shared namespace.

Reservation commands include:

|Command|Purpose|
|---|---|
|Reserveration Register|Register a host reserveration key|
|Reservation Acquire|Acquire or preempt a reservation|
|Reservation Release|Release a reservation|
|Reservation Report|Report current reservation status|

Reservations can enforce access rules such as:

- One writer, many readers.
- Exclusive access.
- Write exclusive access.
- All registrants access.

This is important for clustered filesystems and high-availability storage.

19. Namespace attributes<br>
The Identify Namespace data structure reports many namespace attributes. Important examples include:

|Field|Meaning|
|---|---|
|NSZE|Namespace size in logical blcok|
|NCAP|Namespace capacity|
|NUSE|Namespace utilization|
|NSFEAT|Namespace features|
|NLBAF|Number of supported LBA size|
|FLBAS|Current formatted LBA size|
|MC|Metadata capabilites|
|DPC|Data protection capabilites|
|DPS|Current data protection settings|
|NMIC|Multipath and sharing capabilities|
|RESCAP|Reservation capabilites|
|FPI|Format progress indicator|
|DLFEAT|Deallocated logical block features|
|NAWUN|Namespace atomic write unit normal|
|NAWUPF|Namespace atomic write unit power fail|
|NACWU|Namespace atomic compare and write unit|
|NABSN|Namespace atomic boundary size normal|
|NABO|Namespace atomic boundary offset|
|NABSPF|Namespace atomic boundary size power fail|
|NOIOSB|Namespace optimal I/O boundary|
|NVMCAP|NVM Capacity|
|ANAGRPID|ANAS group identifier|
|NSATTR|Namespace attributes|
|NVMSETID|NVM Set identifier|
|ENDGID|Endurance Group identifier|
|NGUID|Namespace Globally Unique Identifier|
|EUI64|IEEE EUI-64 identifier|

Not every field is meaningful for every device or every command set.

20. Atomic write properties
#atomicity #atomic
Namespaces report atomic write characteristics.

These fields tell the host the maximum amount of data that can be written atomically under certain conditions.

Important fields include:

|Field|Meaning|
|---|---|
|NAWUN|Atomic write unit during normal operation|
|NAWUPF|Atomic write unit during power fail or error conditions|
|NACWU|Atomic compare-and-write unit|
|NABSN|Atomic boundary size normal|
|NABO|Atomic boundary offset|
|NABSPF|Atomic boundary size power fail|

These values are expressed in logical blocks.

They are important for databases, journaling filesystems, and applications that rely on atomic update guarantees.

21. Deallocated or unwritten logical blocks
#deallocation
NVMe namespaces may support deallocation. A host can issue commands such as Dataset Management or Write Zeroes with deallocation attributes, depending on command set support. The namespace reports deallocation behavior through fields such as `DLFEAT`.

A deallocated logical block may read back as:

- Zeroes
- A specific deterministic value
- Indeterminate data

The exact behavior is reported by the namespace. This is relevant for thin provisioning, discard/TRIM, and storage efficiency.

22. Thin provisioning
#thin_provisioning
A namespace may be thin-provisioned. In this case, the namespace’s logical address space may be larger than the physical capacity currently allocated to it.

Example:

```
NSZE = 10 TB logical address space
NCAP = 5 TB allocatable capacity
NUSE = 1 TB currently used
```

The host sees a 10 TB namespace, but the backing storage may allocate physical media only as writes occur. Thin provisioning support is indicated through namespace feature fields.

23. Namespace write protection<br>
NVMe supports namespace write protection mechanisms in some configurations. A namespace may be write protected so that write commands are rejected while reads are still allowed. Write protection can be temporary or persistent depending on controller support and configuration.

This is useful for:

- Firmware or recovery images.
- Secure deployment.
- Preventing accidental modification.
- Compliance-related workflows.

24. Namespace capacity versus NVM capacity<br>
In the Identify Namespace data structure, there may be both logical-block-based and byte-based capacity fields.

Examples:

|Field|Unit|
|---|---|
|NSZE|Logical blocks|
|NCAP|Logical blocks|
|NUSE|Logical blocks|
|NVMCAP|Bytes|

`NVMCAP` represents the total NVM capacity associated with the namespace in bytes. Logical block fields depend on the current or supported LBA format.

25. Namespace changes and notificationsMbr
Namespace configuration can change while a system is running.

Examples:

- Namespace created.
- Namespace deleted.
- Namespace attached.
- Namespace detached.
- Namespace resized.
- Namespace reformatted.
- ANA state changed.
- Namespace attributes changed.

NVMe supports mechanisms for notifying the host, including:

|Mechanism|Purpose|
|---|---|
|Asynchronous Event Request|Notify host of events|
|Changed Namespace List log page|Report namespaces that changed|
|ANA log page|Report path state changes|
|Identify commands|Re-read namespace state|

When the host receives a namespace-related asynchronous event, it should usually re-read Identify data and namespace logs.

26. Namespace management command flow<br>
A simplified namespace creation flow is:

```
1. Host checks controller capabilities with Identify Controller.
2. Host issues Namespace Management Create.
3. Controller allocates namespace and returns an NSID.
4. Host issues Namespace Attachment Attach for one or more controllers.
5. Host identifies the namespace.
6. Host formats the namespace if required.
7. Host begins I/O.
```

Deletion flow:

```
1. Host stops I/O.
2. Host detaches namespace from controllers.
3. Host deletes namespace using Namespace Management Delete.
4. Controller reports namespace inventory change.
```

27. Namespace and NVM Sets<br>
NVMe includes concepts such as NVM Sets and Endurance Groups.

A namespace may be associated with:

|Concept|Meaning|
|---|---|
|NVM Set|A grouping of NVM resources with potentially shared performance or endurance characteristics|
|Endurance Group|A grouping of media with common endurance and wear reporting|

The Identify Namespace data may include:
```
NVMSETID
ENDGID
```
These fields help the host understand placement, isolation, endurance, and performance characteristics.

For example, a storage device may have multiple endurance groups, and a namespace may belong to one of them.

28. Namespace granularity and performance hints<br>
Namespaces can report optimal I/O sizes and boundaries.

Examples include:

|Field|Purpose|
|---|---|
|NPWG|Namespace preferred write granularity|
|NPWA|Namespace preferred write alignment|
|NPDG|Namespace preferred deallocate granularity|
|NPDA|Namespace preferred deallocate alignment|
|NOWS|Namespace optimal write size|
|NOIOSB|Namespace optimal I/O boundary|

These are hints to help the host choose efficient I/O sizes and alignments.

For example, if the namespace prefers writes aligned to a certain boundary, the filesystem or block layer may use that information to improve performance and reduce write amplification.

29. Namespace formatting and format progress<br>
Formatting can take time. The namespace may report format progress using the Format Progress Indicator, or `FPI`.

This lets the host determine whether a namespace format operation is in progress and how far it has advanced.

During formatting, I/O behavior depends on controller capabilities and command rules.

30. Common Identify commands related to namespaces<br>
The Identify admin command is central to namespace discovery.

Common namespace-related Identify operations include:

|Identify Operation|Purpose|
|---|---|
|Identify|Return namespace attributes|
|Identify Namespace ID List|Return active namespace IDs|
|Identify Namespace Identification Descriptor List|Return persistent namespace identifers|
|Identify Controller List attached to namespace|Return controllers attached to a namespace|
|Identify Namespace List, allocated or active|Discover namespace inventory|
|Identify I/O Command Set Data|Determin command-set-specific support|

Exact CNS values and returned structures depend on the NVMe revision and enabled command sets.

31. Error behavior for invalid namespaces<br>
If a host submits an I/O command to an invalid, inactive, detached, or unsupported namespace, the controller returns an error status.

Typical reasons include:

- NSID does not exist.
- Namespace exists but is not attached to the controller.
- Namespace is currently unavailable.
- Namespace is formatted with an unsupported command set.
- Namespace is in a transitional state.
- Namespace access is blocked by reservation rules.

The host should not send normal I/O to namespaces that are not reported as active for that controller.

32. Namespace compared with partitions
#partition
A namespace is not the same thing as a partition. A namespace is created and exposed by the NVMe subsystem. It appears to the host operating system as a block device. A partition is created by the host inside a block device.

Example:
```
NVMe Namespace 1
 ├── GPT partition 1
 ├── GPT partition 2
 └── GPT partition 3
```
So an NVMe SSD may have:
```
/dev/nvme0n1
```
And the operating system may create partitions:
```
/dev/nvme0n1p1
/dev/nvme0n1p2
```
Here:
```
nvme0n1  -> NVMe namespace
p1/p2    -> host-created partitions
```

33. Practical Linux example<br>
On Linux, namespaces commonly appear as devices like:
```
/dev/nvme0n1
/dev/nvme0n2
/dev/nvme1n1
```
Meaning:

| Device  | Meaning         |
| ------- | --------------- |
| `nvme0` | NVMe Controller |
| `n1`    | Namespace 1     |
| `n2`    | Namespace 2     |
Example command:

```
nvme list
```

Example namespace identify command:

```
nvme id-ns /dev/nvme0n1
```

Example namespace list:

```
nvme list-ns /dev/nvme0
```

On a namespace-management-capable device, commands may exist to create, attach, detach, or delete namespaces, though many consumer drives do not support those operations.

34. Important points to remember<br>
1. A namespace is the NVMe logical storage object used for I/O.
2. Each namespace is addressed by an NSID.
3. A namespace may be private or shared.
4. A namespace may be attached to one or more controllers.
5. Namespace identity is better determined using NGUID, EUI-64, UUID, or 6. namespace identification descriptors, not only NSID.
6. Namespaces have size, capacity, utilization, LBA format, metadata, and protection attributes.
7. Namespace management allows creation, deletion, attachment, and detachment.
8. NVMe 2.x separates the base architecture from command-set-specific namespace behavior.
9. Shared namespaces may use ANA and reservations for multipath and coordinated access.
10. A namespace is not a partition; partitions are created by the host inside a namespace.


