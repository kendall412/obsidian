
## Detailed NVMe Compare-and-Write Fused Operation

> The **Compare + Write fused operation** is NVMe's mechanism for implementing an atomic **Compare-And-Swap (CAS)** style update.

> In a fused **Compare + Write** operation, the **Compare command compares the data currently stored at the specified LBAs with a data buffer supplied by the host**. It does **not** compare metadata, command IDs, timestamps, or version numbers unless those are part of the actual user data stored in the blocks.

This prevents [[race conditions]] and lost updates.

## What Exactly Is Compared?

Suppose:

```
LBA 100
Size = 4096 bytes
```

The host provides a buffer containing the **expected contents** of that 4096-byte block.

### Compare Command

```
Opcode = Compare
SLBA   = 100
NLB    = 0   (1 logical block)
```

PRP/SGL points to:

```
Host Memory
+----------------+
| Expected Data  |
+----------------+
```

Controller reads:

```
SSD Data at LBA 100
```

and performs conceptually (C):

```
memcmp(ssd_data, host_expected_data, 4096)
```

## Example: Compare Succeeds

### Host Expected Buffer

```
Byte 0-15

01 02 03 04
AA BB CC DD
11 22 33 44
55 66 77 88
```

### Data Stored in SSD

```
01 02 03 04
AA BB CC DD
11 22 33 44
55 66 77 88
```

Result:

```
Compare = SUCCESS
```

Write command is allowed to execute.

## Example: Compare Fails

### Host Expected Buffer

```
01 02 03 04
AA BB CC DD
11 22 33 44
55 66 77 88
```

### SSD Data

```
01 02 03 04
AA BB CC DD
11 22 33 99
55 66 77 88
```

One byte differs:

```
44 != 99
```

Result:

```
Compare = FAILED
```

The fused Write is aborted.

## Database Example

Suppose a database record stored in LBA 100 is:

```
Account Balance = $100
```

Host reads it and later wants to update it to:

```
Account Balance = $80
```

The fused pair works like:

### Compare Command

```
Expected:
Balance = $100
```

### Write Command

```
New:
Balance = $80
```

Controller does:

```
Current data == $100 ?
```

If yes:

```
Write $80
```

If no:

```
Abort
```

This ensures nobody modified the record after it was read.

## What If Multiple LBAs Are Involved?

Suppose:

```
SLBA = 100
NLB  = 7
```

This means 8 logical blocks are involved.

For a 4 KiB LBA format:

```
8 × 4096 = 32768 bytes
```

The controller compares the entire 32 KiB range.

Conceptually (C):

```
memcmp(
    ssd_data[32768],
    host_expected_data[32768],
    32768
)
```

Any mismatch causes failure.

## Does It Compare NAND Data Directly?

Not necessarily.

The controller compares against the **current logical block contents**.

Internally it may obtain that data from:

- DRAM cache
- Write buffer
- Read cache
- NAND flash

depending on where the latest version resides.

The host doesn't care where it comes from; it only cares about the logical contents of the LBA.

## Why This Is Useful

The Compare command effectively asks:

```
"Is the data still exactly what I think it is?"
```

The subsequent Write says:

```
"If yes, replace it with this new data."
```

Together, the fused operation implements the storage equivalent of a CPU's atomic compare-and-swap:

```
if (memory == expected)
    memory = new_value;
else
    fail;
```

That is the fundamental purpose of the NVMe fused Compare + Write operation.


---
# Example Scenario

Suppose LBA 100 currently contains:

```
Version = 5
```

Host wants to update it to:

```
Version = 6
```

but only if nobody else has modified it.

# Step 1: Host Builds Compare Command

The host creates a Compare command:

```
Opcode = Compare
SLBA   = 100
NLB    = 1
FUSE   = First Command (01b)
```

Data buffer contains:

```
Expected Data = Version 5
```

Meaning:

```
Verify that LBA 100 still contains Version 5.
```

# Step 2: Host Builds Write Command

Immediately after the Compare command in the same Submission Queue:

```
Opcode = Write
SLBA   = 100
NLB    = 1
FUSE   = Second Command (10b)
```

Data buffer contains:

```
New Data = Version 6
```

Meaning:

```
If Compare succeeds,
write Version 6.
```

# Submission Queue Layout

The queue might look like:

```
SQ Head
  |
  V
+---------+
| Compare |
+---------+
| Write   |
+---------+
| Read    |
+---------+
| Read    |
+---------+
```

Notice:

```
Compare and Write are adjacent.
```

This is mandatory.

# Step 3: Host Rings Doorbell

Host updates SQ Tail:

```
SQT = SQT + 2
```

and writes the Submission Queue Tail Doorbell.

```
Host --> Controller
       "Two new commands available"
```

# Step 4: Controller Fetches Commands

Using DMA, the controller fetches:

```
Command #1 = Compare
Command #2 = Write
```

and notices:

```
FUSE = 01b
FUSE = 10b
```

which means:

```
These form a fused pair.
```

The controller now treats them as a single atomic transaction.

# Internal Controller Validation

Before execution the controller verifies:

### Rule 1

First command:

```
FUSE = 01b
```

### Rule 2

Second command:

```
FUSE = 10b
```

### Rule 3

Commands are adjacent.

### Rule 4

Same namespace.

### Rule 5

Compatible LBA ranges.

If any rule fails:

```
Status = Invalid Fused Operation
```

and execution stops.

# Step 5: Controller Locks the Target Range

Internally the controller prevents interference.

Conceptually:

```
Lock LBA 100
```

Real controllers may use:

- Lock tables
- Metadata ownership bits
- Scheduler reservations
- Internal serialization

The exact mechanism is vendor-specific.

Goal:

```
No other command may modify
this LBA between Compare and Write.
```

# Step 6: Execute Compare

Controller reads current data.

```
Flash
  |
  v
LBA 100
```

Current value:

```
Version 5
```

Expected value from host:

```
Version 5
```

Controller performs:

```
memcmp(current, expected)
```

# Compare Success Case

```
Current = Version 5
Expected = Version 5
```

Match.

Result:

```
Compare Success
```

Controller proceeds to Write.

# Step 7: Execute Write

Controller writes:

```
Version 6
```

to LBA 100.

Internally:

```
Write Command
    |
FTL Mapping Update
    |
NAND Program
    |
Metadata Commit
```

After completion:

```
LBA 100 = Version 6
```

# Step 8: Generate Completions

Even though execution is atomic:

```
Compare
+
Write
```

each command receives its own CQE.

Completion Queue:

```
+------------------+
| Compare Success  |
+------------------+
| Write Success    |
+------------------+
```

# Failure Case

Suppose another host already updated the block.

Current contents:

```
Version 7
```

Host expects:

```
Version 5
```

Controller compares:

```
Version 7
vs
Version 5
```

Mismatch.

Result:

```
Compare Failure
```

# What Happens to Write?

The Write command is never executed.

Controller aborts it.

CQEs:

```
+------------------+
| Compare Failed   |
+------------------+
| Write Aborted    |
+------------------+
```

Data remains:

```
Version 7
```

# Timeline Without Fuse

```
Time -->

Host A:
Read V5
            Write V6

Host B:
      Write V7
```

Result:

```
V6 overwrites V7
```

Lost update.

# Timeline With Fuse

```
Time -->

Host A:
Compare V5
Write V6

Host B:
      Write V7
```

Controller serializes access:

```
Compare V5
   |
Match?
   |
Yes
   |
Write V6
```

or

```
Compare V5
   |
Match?
   |
No
   |
Abort Write
```

No lost update occurs.

# Controller View

Internally the fused pair behaves like:

```
Atomic Transaction
+---------------------+
| Compare LBA 100     |
| Write LBA 100       |
+---------------------+
```

The controller scheduler cannot insert another command between them.

Normal commands:

```
Read
Write
Flush
Read
```

may be reordered.

A fused pair cannot be split.

# Why Databases Like This

Databases frequently need (C):

```
if (record == expected)
    record = new_value;
else
    fail;
```

Examples:

- Lock acquisition
- Transaction metadata
- B-tree node updates
- Version counters
- Distributed consensus state

Compare+Write fused operations allow the storage device itself to perform this atomically.

# Complete Flow Diagram

```
Host
 |
 |-- Compare (FUSE=01)
 |     Expected = Version 5
 |
 |-- Write (FUSE=10)
 |     New = Version 6
 |
 v
Submission Queue
 |
 v
NVMe Controller
 |
 |-- Validate fused pair
 |-- Lock target LBA
 |-- Read current data
 |-- Compare with expected
 |
 |-- Match?
 |      |
 |      +-- No --> Abort Write
 |      |
 |      +-- Yes --> Execute Write
 |
 v
Completion Queue
 |
 +-- Compare CQE
 +-- Write CQE
```

The key concept is that the controller guarantees **the compare result and the subsequent write occur as one indivisible operation**, preventing any other command from modifying the target logical blocks between those two steps.