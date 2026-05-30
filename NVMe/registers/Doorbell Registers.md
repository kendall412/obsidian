
> A **doorbell register** is a memory-mapped register that the host writes to in order to notify the NVMe controller that something has changed in a queue.

Think of it as literally "ringing a doorbell":

- Host places commands into a Submission Queue (SQ)
- Host rings the Submission Queue Doorbell
- NVMe controller notices the new commands and processes them

Likewise:

- Controller places completions into a Completion Queue (CQ)
- Host processes those completions
- Host rings the Completion Queue Doorbell
- Controller knows those CQ entries have been consumed

# Why Doorbells Exist

Submission and Completion Queues reside in host memory.

The NVMe controller does **not continuously scan host memory** because that would waste PCIe bandwidth and controller resources.

Instead:

1. Host updates queue entries in memory.
2. Host writes a small value to a doorbell register.
3. Controller immediately knows where new work exists.

This provides:

- Low latency
- Low overhead
- Efficient PCIe operation

# Where Doorbells Are Located

Doorbells are [[Memory-Mapped IO (MMIO)]] registers located in the controller's PCIe BAR.

Usually:

- BAR0 contains NVMe registers
- Doorbells begin at offset:

1000<sub>h</sub>

from the NVMe register space.

# Doorbell Register Layout

For each queue pair:

```
SQ0 Tail Doorbell
CQ0 Head Doorbell

SQ1 Tail Doorbell
CQ1 Head Doorbell

SQ2 Tail Doorbell
CQ2 Head Doorbell
...
```

Each queue gets:

- One Submission Queue Doorbell
- One Completion Queue Doorbell

---
# Queue Example

Assume:

```
SQ Size = 64 entries
```

Current state:

```
SQ Tail = 10
SQ Head = 10
```

Queue is empty.

Host submits a command:

```
SQ[10] = READ
Tail = 11
```

Host then writes:

```
SQ0 Doorbell = 11
```

Controller sees:

```
New commands exist betweenold tail and new tail
```

and starts processing.

# Submission Queue Doorbell (SQTDBL)

The Submission Queue Tail Doorbell tells the controller:

> "I have added commands up to this tail index."

Example:

```
Queue depth = 64

Tail moves:
10 → 11
```

Host writes:

```
SQTDBL = 11
```

Controller begins fetching commands.

# Completion Queue Doorbell (CQHDBL)

The Completion Queue Head Doorbell tells the controller:

> "I have consumed completion entries up to this head index."

Example:

Controller generates completions:

```
CQ[0]
CQ[1]
CQ[2]
```

Host reads them.

Host advances:

```
Head = 3
```

Host writes:

```
CQHDBL = 3
```

Controller now knows those entries may be reused.

# Doorbell Register Size

Each doorbell is:

```
32 bits
```

wide.

Only the lower bits are used according to queue size.

# Doorbell Stride (DSTRD)

The spacing between doorbell registers is defined by:

```
CAP.DSTRD
```

field in the Controller Capabilities register.

Formula:

Doorbell Offset = 1000<sub>h</sub> + (2 + _qid_ x 4 x 2<sup>DSTRD</sup>)

where:

- _qid_ = queue identifier
- DSTRD = doorbell stride

---
# Example (Most Common)

Most SSDs use:

```
DSTRD = 0
```

Therefore:

```
Doorbell spacing = 4 bytes
```

Layout:

|Offset|Register|
|---|---|
|1000h|SQ0 Tail|
|1004h|CQ0 Head|
|1008h|SQ1 Tail|
|100Ch|CQ1 Head|
|1010h|SQ2 Tail|
|1014h|CQ2 Head|

# Host Read Command Example

## 1. Host creates command

```
SQ1[5] = READ LBA 100
```

## 2. Tail advances

```
Tail = 6
```

## 3. Ring doorbell

```
write 6 → SQ1TDBL
```

PCIe MMIO write:

```
BAR0 + 1008h = 6
```

## 4. Controller receives notification

```
New SQ entries available
```

## 5. Controller executes command

Reads NAND data.

## 6. Controller posts completion

```
CQ1[3]
```

## 7. Host reads completion

```
Head = 4
```

## 8. Host rings CQ doorbell

```
write 4 → CQ1HDBL
```

Controller may now reuse CQ entry 3.

# Relationship to Queue Pointers

Each queue maintains:

## Submission Queue

```
Head -> maintained by controller
Tail -> maintained by host
```

Doorbell updates:

```
Host writes Tail
```

to notify controller.

## Completion Queue

```
Head -> maintained by host
Tail -> maintained by controller
```

Doorbell updates:

```
Host writes Head
```

to notify controller.

# Why Doorbells Are Critical

Without doorbells:

- Controller would have to constantly poll host memory.
- PCIe traffic would increase dramatically.
- Latency would increase.

Doorbells provide:

- Event-driven operation
- Very low command latency
- High IOPS scalability
- Efficient queue management

# Visual Summary

```
Host CPU
   |
   | 1. Place command in SQ
   v
+-------------------+
| Submission Queue  |
+-------------------+
          |
          | 2. Write SQ Tail Doorbell
          v
+-------------------+
| NVMe Controller   |
+-------------------+
          |
          | 3. Execute command
          |
          v
+-------------------+
| Completion Queue  |
+-------------------+
          ^
          |
          | 4. Host reads completion
          |
          | 5. Write CQ Head Doorbell
          |
Host CPU
```

In NVMe, the **doorbell registers are the synchronization mechanism between host software and the controller**, allowing queue updates to be communicated through lightweight MMIO writes instead of expensive memory polling.