
> The interrupt is **not before** the Completion Queue doorbell update. The interrupt tells the host that new completions exist, and then the host processes those completions and updates the Completion Queue Head Doorbell afterward.
# Completion Flow Overview

Assume the controller finishes a Read command.

```
Host                NVMe Controller

Read Command
   |
   |-------------------->
   |
   |              Execute Read
   |                     |
   |                     v
   |              Write CQ Entry
   |                     |
   |                     v
   |              Generate Interrupt
   |<--------------------
   |
Interrupt Handler
   |
Read CQ Entry
   |
Update CQ Head
   |
Write CQ Head Doorbell
   |-------------------->       
```

# Why Does the Controller Generate an Interrupt?

When the controller completes a command, it writes a Completion Queue Entry (CQE):

```
CQ

+---------+
| CQE #1  |
+---------+
| CQE #2  |
+---------+
| CQE #3  |
+---------+
```

The host doesn't continuously read the CQ because that would waste CPU cycles.

Instead, the controller says:

```
"I've placed new completions in the CQ."
```

via:

- [[Message Signaled Interrupts eXtended (MSI-X)]] interrupt (most common)
- MSI
- Legacy INTx

# What Happens After the Interrupt?

The interrupt handler runs.

Example:

```
CQ Head = 10
CQ Tail = 13
```

Meaning:

```
Entries 10,11,12 are new completions
```

Host processes:

```
CQE 10
CQE 11
CQE 12
```

# Why Update the CQ Head Doorbell?

The controller knows where it writes completions. The controller does **not** know which completions the host has consumed. Therefore the host must tell the controller:

```
"I have processed completions up to CQ Head = 13."
```

by writing:

```
CQ Head Doorbell
```

# Example

Initially:

```
CQ Depth = 16

Head = 4
Tail = 4
```

Queue empty.

Controller completes three commands:

```
Head = 4
Tail = 7
```

```
+----+----+----+
|CQE4|CQE5|CQE6|
+----+----+----+
```

Controller generates interrupt.


Host receives interrupt:

```
Process CQE4
Process CQE5
Process CQE6
```

Now:

```
Head = 7
```

Host writes:

```
CQ Head Doorbell = 7
```

to tell controller:

```
These entries may now be reused.
```

# Why Not Update the Doorbell First?

Suppose:

```
Interrupt arrives
```

Host immediately writes:

```
CQ Head Doorbell = 7
```

before reading completions.
Controller now thinks:

```
Entries 4,5,6 are free.
```

It may overwrite them.
Host could lose completions.

Therefore:

```
Read CQEs
    ↓
Process CQEs
    ↓
Advance CQ Head
    ↓
Write CQ Head Doorbell
```

is mandatory.

# Visual Timeline

### Correct Sequence

```
Controller
    |
    | Write CQE
    |
    | Interrupt
    V

Host
    |
    | Read CQE
    | Process CQE
    | Advance Head
    |
    | Write CQ Head Doorbell
    V

Controller
    |
    | Reuse CQ entries
```

### Incorrect Sequence

```
Controller
    |
    | Write CQE
    | Interrupt
    V

Host
    |
    | Write Doorbell
    |
    | Read CQE
```

Potential problem:

```
Controller overwrites CQE
before host reads it.
```

# Why Does NVMe Use This Design?

The Completion Queue resides in **host memory**, not controller memory.

```
Host RAM

+-------------------+
| Completion Queue  |
+-------------------+
```

The controller can write completions into it, but it cannot know when software has finished processing them.
Therefore NVMe uses a producer-consumer model:
### Controller (Producer)

```
Writes CQEs
Advances Tail
Generates Interrupt
```

### Host (Consumer)

```
Reads CQEs
Advances Head
Rings CQ Head Doorbell
```

The CQ Head Doorbell is essentially an acknowledgment:

```
"I have consumed these completion entries."
```

# Interrupt Coalescing Note

In high-performance systems, the controller may complete many commands before generating an interrupt.

Example:

```
Complete 50 commands
      ↓
Generate one interrupt
```

Host then processes all 50 CQEs and updates the CQ Head Doorbell once.

This reduces interrupt overhead while maintaining the same ordering:

```
Completion(s)
      ↓
Interrupt
      ↓
Host processes CQEs
      ↓
CQ Head Doorbell update
```

> So the interrupt comes **after the controller writes completion entries**, and the **CQ Head Doorbell update comes after the host consumes those completion entries**. The doorbell is effectively the host's acknowledgment that the completion queue entries can be reused.

