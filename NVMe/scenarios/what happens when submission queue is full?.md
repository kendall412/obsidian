
> When an NVMe **Submission Queue (SQ)** is full, the host cannot post any more commands into that queue until the controller processes some existing commands and advances the queue head.

## How NVMe Detects a Full Queue

Each Submission Queue has:

- **Tail Pointer (SQT)** — maintained by the host
- **Head Pointer (SQH)** — maintained by the controller

Example for a queue of depth 8:

```
Entry:  0 1 2 3 4 5 6 7
```

Initially:

```
SQH = 0
SQT = 0
```

After the host submits 7 commands:

```
SQH = 0
SQT = 7
```

The queue is considered full when advancing the tail would make it equal to the head.

To distinguish between:

- Empty queue
- Full queue

NVMe intentionally leaves **one entry unused**.

Thus:

```
Usable entries = Queue Depth - 1
```

For a queue depth of 64:

```
Maximum outstanding commands = 63
```

## What Happens Next?

### Scenario

Queue depth = 64

Current state:

```
Outstanding commands = 63
SQ is full
```

The host wants to submit another Read command.

It cannot write the command because doing so would overwrite an unprocessed command.

The host must wait.

## Host Driver Behavior

A well-written NVMe driver will:

1. Check available SQ entries.
2. Detect that the queue is full.
3. Stop posting new commands.
4. Wait for completions.
5. Reclaim SQ entries.
6. Submit additional commands.

Conceptually:

```
while (SQ full)
    wait for completions

submit new command
```

The host never intentionally overwrites a valid SQ entry.

## How Space Becomes Available

Suppose the queue is full:

```
SQH = 10
SQT = 9
```

Controller processes command at entry 10.

Then:

```
SQH = 11
```

Now one slot is free.

The controller typically reports the new SQ Head Pointer in the Completion Queue entry.

Example completion:

```
Completion:
    Command ID = 123
    SQ Head = 11
```

The host learns:

> "The controller has consumed everything up to entry 10."

Now the host can reuse entry 10.

## Doorbell Interaction

The host normally:

1. Writes commands into SQ memory.
2. Updates the Submission Queue Tail Doorbell.

Example:

```
Write command to SQ[15]
Ring SQT doorbell = 16
```

If the queue is full:

```
No free SQ entry exists
```

Therefore:

```
No command written
No doorbell rung
```

The host must wait.

## Example Timeline

Queue depth = 8

Usable entries = 7

### Step 1

```
SQH = 0
SQT = 0
Outstanding = 0
```

### Step 2

Host submits 7 commands:

```
SQH = 0
SQT = 7
Outstanding = 7
```

Queue full.

### Step 3

Host wants to submit command #8

```
Cannot submit.
Must wait.
```

### Step 4

Controller executes one command.

```
SQH = 1
SQT = 7
Outstanding = 6
```

One slot becomes available.

### Step 5

Host submits another command.

```
SQH = 1
SQT = 0   (wrap-around)
Outstanding = 7
```

Queue full again.

## Does the Controller Reject the Command?

Normally, no.

A properly functioning NVMe driver should never ring the doorbell for more commands than the queue can hold.

The queue-full condition is handled by the host driver before submission.

So the controller usually never sees an invalid "queue overflow" command.

## Effect on Performance

When the SQ stays full:

```
iodepth ≈ queue depth
```

the SSD is heavily loaded.

This is often desirable in benchmarks because it maximizes:

- IOPS
- NAND parallelism
- PCIe utilization

However, latency generally increases because new commands must wait for free SQ entries.

## Real Example

A common Linux NVMe configuration might be:

```
Submission Queue Depth = 1024
```

Maximum usable entries:

```
1023
```

If a workload such as fio runs (bash):

```
fio --iodepth=1023
```

the queue may remain nearly full all the time. The NVMe driver continuously waits for completions and immediately reuses freed entries to maintain the requested I/O depth.

### Key Point

A full Submission Queue does **not** mean the SSD stops working. It means:

> The host has already given the controller the maximum number of outstanding commands the queue can hold, and must wait for completions before submitting more commands.