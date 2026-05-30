
In NVMe, a **Command Set** is a collection of commands and data structures that define how a host communicates with a namespace. It specifies the operations supported by a storage device and the format of the commands used to perform them.

Think of it as a **language** spoken between the host and the NVMe controller.

## Why NVMe Uses Command Sets

Originally, NVMe supported only one command set:

- **NVM Command Set** (for NAND flash SSDs)

As NVMe evolved, it became a general-purpose storage protocol capable of supporting different storage technologies. To accommodate this, the NVMe specification introduced **multiple command sets**.

This allows one NVMe controller to support different storage types while using the same transport (PCIe, TCP, RDMA, etc.).

---

## Main NVMe Command Sets

### 1. NVM Command Set

The traditional SSD command set used by almost all NVMe SSDs today.

Common I/O commands:

|Opcode|Command|
|---|---|
|00h|Flush|
|01h|Write|
|02h|Read|
|04h|Write Uncorrectable|
|05h|Compare|
|08h|Write Zeroes|
|09h|Dataset Management|
|0Dh|Reservation Register|
|11h|Reservation Acquire|
|15h|Reservation Release|
|18h|Copy|

Typical operations:

- Read LBAs
- Write LBAs
- Flush cache
- Trim/Deallocate blocks
- Reservations

---

### 2. Zoned Namespace (ZNS) Command Set

Designed for zoned storage devices.

A namespace is divided into zones.

Additional commands:

|Command|
|---|
|Zone Append|
|Zone Open|
|Zone Close|
|Zone Reset|
|Zone Finish|

Benefits:

- Lower write amplification
- Better endurance
- Host-controlled data placement

Used in:

- Large-scale cloud storage
- Data centers

---

### 3. Key Value (KV) Command Set

Stores data as:

```
<Key, Value>
```

instead of:

```
LBA → Block
```

Commands include:

- Store
- Retrieve
- Delete
- Iterate

Example:

```
Key: User123Value: Customer record
```

Not widely deployed but useful for databases.

---

## How the Host Knows Which Command Set Is Used

The host issues an Identify command.

The controller reports:

- Supported I/O Command Sets (IOCS)
- Namespace Command Set Identifier (CSI)

Example:

|CSI|Command Set|
|---|---|
|00h|NVM|
|02h|Zoned Namespace|
|01h|Key Value|

The host then uses the appropriate commands for that namespace.

---

## Command Set vs Admin Commands

A common source of confusion:

### Admin Command Set

These commands manage the controller itself:

Examples:

- Identify
- Get Log Page
- Create SQ
- Create CQ
- Firmware Download
- Firmware Commit
- Format NVM

These exist regardless of the I/O command set.

---

### I/O Command Set

These commands perform data operations:

For NVM:

```
ReadWriteFlushCompare
```

For ZNS:

```
Zone AppendZone ResetZone Open
```

For KV:

```
StoreRetrieveDelete
```

---

## Command Set Architecture

```
Host Driver    |    v+----------------------+| NVMe Admin Commands  |+----------------------+           |           v+----------------------+| Command Set Layer    |+----------------------+     /        |        \    /         |         \ NVM        ZNS         KV(Read)   (Zones)   (Key/Value)
```

The Admin Queue manages the controller, while I/O Queues carry commands belonging to a specific command set.

---

## Example: Read Command in the NVM Command Set

When the host wants to read data:

1. Build a Read SQE.
2. Set:
    - Opcode = 02h (Read)
    - NSID
    - Starting LBA
    - Number of Logical Blocks
3. Place SQE in Submission Queue.
4. Ring Submission Queue Doorbell.
5. Controller executes command.
6. Completion entry appears in Completion Queue.

The meaning of Opcode `02h` comes from the **NVM Command Set specification**.

---

## Relationship to Namespace

Each namespace can be associated with a command set.

Example:

|Namespace|Command Set|
|---|---|
|NSID 1|NVM|
|NSID 2|ZNS|
|NSID 3|KV|

The host checks the namespace's CSI value to determine which commands are valid.

---

### Summary

A **Command Set** in NVMe is the specification that defines the valid I/O commands, command formats, and behaviors for a namespace.

Key command sets are:

|Command Set|Purpose|
|---|---|
|NVM|Traditional SSD block storage|
|ZNS|Zoned storage|
|KV|Key-value storage|

Admin commands are separate and always available, while I/O commands depend on the command set assigned to the namespace.