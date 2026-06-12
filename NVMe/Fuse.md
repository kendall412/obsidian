
> In NVMe, **Fuse** (or _Fused Operation_) is a mechanism that allows **two commands to be executed atomically as a pair**. The controller guarantees that either:

1. Both commands complete successfully in order, or
2. Neither command is performed.

## Why Fused Commands Are Needed in NVMe

The primary reason for fused commands is to provide **atomic operations** in a highly parallel storage system.

NVMe is designed to support:

- Multiple Submission Queues (SQs)
- Thousands of outstanding commands
- Multiple CPU cores issuing I/O simultaneously
- Out-of-order command execution by the controller

Without a mechanism like fused commands, certain operations would be vulnerable to [[race conditions]].

This is primarily used for operations that require a **read-modify-write** sequence without interference from other commands.

## Where is Fuse Defined?

The fuse information is contained in the **FUSE field** of the NVMe command capsule.

In Command Dword 0 (CDW0):

```
31               16 15    14 13   10 9      8 7      0
+----------------+--------+--------+---------+--------+
| Command ID     | FUSE   | Reserved | PSDT  | OPC    |
+----------------+--------+--------+---------+--------+
```

The FUSE field is 2 bits wide.

### FUSE Values

| Value | Meaning                      |
| ----- | ---------------------------- |
| 00b   | Normal command (not fused)   |
| 01b   | First command of fused pair  |
| 10b   | Second command of fused pair |
| 11b   | Reserved                     |
## Why Fused Commands Exist

Suppose a host wants to:

1. Compare a block's current contents.
2. If it matches expected data, write new data.

Without fused commands:

```
Compare
(other host command may occur here)
Write
```

Another command could modify the data between the [[Compare and Write]].

Fused operations prevent this race condition.

## Most Common Example: Compare + Write

### First Command

```
Compare
FUSE = 01b
```

The controller compares data in NAND against host-provided data.

### Second Command

```
Write
FUSE = 10b
```

The controller writes new data only if the Compare succeeds.

Together they behave as one atomic transaction.

## Submission Queue Example

Suppose SQ contains:

|SQ Entry|Command|FUSE|
|---|---|---|
|0|Compare LBA 100|First (01b)|
|1|Write LBA 100|Second (10b)|
|2|Read LBA 200|Normal|

The controller sees:

```
SQ Head
  |
  v
+---------+
| Compare |
+---------+
| Write   |
+---------+
| Read    |
+---------+
```

The Compare and Write are treated as a single atomic operation.

## Controller Rules

The two fused commands must:

- Be adjacent in the same Submission Queue.
- Be submitted in the correct order.
- Use matching LBAs and namespace as required by the specification.
- Not be separated by another command.

Invalid example:

```
Compare (First)
Read
Write (Second)
```

The controller will return an error.

## Completion Behavior

Each command still receives its own Completion Queue Entry (CQE).

Example:

```
CQE #1 -> Compare Success
CQE #2 -> Write Success
```

If the Compare fails:

```
CQE #1 -> Compare Failed
CQE #2 -> Aborted
```

The Write is not executed.

## Atomicity Guarantee

For a fused Compare+Write pair:

```
Old Data
    |
Compare
    |
Success?
    |
   Yes
    |
Write New Data
```

No other command is allowed to access the targeted logical blocks between the fused commands.

This provides an atomic **compare-and-swap** style operation, which is useful for databases, file systems, locks, and metadata updates.

## NVMe Command Flow Example

```
Host
 |
 |-- SQ Entry 1:
 |   Compare LBA 100
 |   FUSE = First
 |
 |-- SQ Entry 2:
 |   Write LBA 100
 |   FUSE = Second
 |
 v
NVMe Controller
 |
 |-- Execute Compare
 |-- If match:
 |      Execute Write
 |-- Else:
 |      Abort Write
 |
 v
Completion Queue
```

### Key Takeaway

**Fuse in NVMe is an atomic command-pair mechanism.** The most common use is **Compare + Write**, where:

- First command: `FUSE = 01b`
- Second command: `FUSE = 10b`
- Commands must be adjacent in the same Submission Queue.
- The controller guarantees atomic execution of the pair.

