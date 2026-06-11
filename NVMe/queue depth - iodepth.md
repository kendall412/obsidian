
## Queue Depth (NVMe Queue Depth; Device/OS Level)

> Queue depth is a **hardware/protocol property** of an NVMe Submission Queue (SQ). It defines the **maximum number of commands that can be stored in the queue at one time**.

 What it is: The actual number of I/O commands waiting in the queue for the storage controller or disk to execute, a metric often seen in system monitoring tools like iostat (e.g., avgqu-sz).
- Relationship to iodepth: The iodepth set in fio directly influences the queue depth seen at the OS and device level, but it's not always a perfect one-to-one match due to OS/driver overhead or limitations.
- Example: Setting `iodepth=64` in fio tells the tool to try and keep 64 requests outstanding, which the OS and storage device then try to process in parallel. 

For example:

- Submission Queue size = 64 entries
- Queue Depth = 64

This means the host can place up to 64 commands into that SQ before it becomes full.

Example SQ:

|Entry|Command|
|---|---|
|0|Read|
|1|Read|
|2|Write|
|...|...|
|63|Read|

Maximum commands = 64

So:

```
Queue Depth = SQ Size
```

The queue depth is configured when the queue is created using the Create I/O Submission Queue command.

## I/O Depth (iodepth; Application Level)

> I/O depth is a **runtime workload characteristic**. It represents the number of outstanding (uncompleted) I/O commands at a given time. `iodepth` in benchmarking tools like fio sets the number of simultaneous I/O requests the application sends to the storage, representing the application's perspective on command queue depth, while "queue depth" generally refers to the total outstanding I/O the device or OS level handles, with `iodepth` directly influencing device queue depth, especially with direct I/O, to measure raw storage performance by saturating the drive's ability to process parallel commands. 

- What it is: A parameter in benchmarking tools (like fio) that defines how many I/O operations are kept "in flight" (pending) by the test application for a given file or device.
- Purpose: To simulate real-world application concurrency and stress the storage device by sending many requests at once, bypassing kernel caching when direct=1 is used.
- Impact: A higher `iodepth` (e.g., 32, 64, 128) allows modern SSDs and NVMe drives to achieve much higher IOPS and throughput because they excel at parallel processing. 


Example:

Queue depth = 64

Host submits:

- Read #1
- Read #2
- Read #3
- Read #4

None have completed yet.

Current:

```
iodepth = 4
```

When one completes:

```
iodepth = 3
```

If the host keeps 32 requests outstanding:

```
iodepth = 32
```

## Example

Assume:

```
Queue Depth = 64
```

Timeline:

### t0

No commands

```
iodepth = 0
```

### t1

Host submits 8 reads

```
iodepth = 8
```

### t2

NVMe completes 3 reads

```
iodepth = 5
```

### t3

Host submits 20 more

```
iodepth = 25
```

### t4

Host fills queue

```
iodepth = 64
```

Now:

```
iodepth = queue depth
```

Queue is full.

## Relationship

The important rule is:

```
iodepth ≤ queue depth
```

You cannot have more outstanding commands than the queue can hold.

For example:

|Queue Depth|Possible iodepth|
|---|---|
|64|0–64|
|128|0–128|
|1024|0–1024|

## Multiple Queues

NVMe supports many Submission Queues.

Example:

- SQ1 depth = 64
- SQ2 depth = 64
- SQ3 depth = 64
- SQ4 depth = 64

Total outstanding commands:

```
64 + 64 + 64 + 64= 256
```

So the system-wide I/O depth can be larger than the depth of any individual queue.

## In fio Benchmarks

When using the fio benchmark tool:

```
fio --iodepth=32
```

means:

> Keep 32 I/O requests outstanding at all times.

The tool continuously submits new commands as completions arrive so that approximately 32 requests remain in flight.

Examples:

|fio iodepth|Meaning|
|---|---|
|1|One request at a time|
|4|Four outstanding requests|
|32|Thirty-two outstanding requests|
|128|One hundred twenty-eight outstanding requests|

This is a workload parameter, not a queue size parameter.

## NVMe Performance Impact

### Low iodepth

```
iodepth = 1
```

Process:

1. Submit command
2. Wait
3. Complete
4. Submit next

Performance is limited by latency.

### High iodepth

```
iodepth = 64
```

Process:

1. Submit 64 reads
2. Controller works on many simultaneously
3. Completions arrive continuously

This allows the SSD to exploit internal parallelism:

- Multiple NAND dies
- Multiple channels
- Multiple planes

Result:

- Higher IOPS
- Better throughput

## Simple Analogy

Think of a restaurant:

- **Queue depth** = number of orders the kitchen can hold on its board.
- **iodepth** = number of orders currently waiting to be cooked.

Example:

```
Kitchen board capacity = 100 orders
(queue depth = 100)

Current orders waiting = 20
(iodepth = 20)
```

The board can hold 100, but only 20 are currently outstanding.

### Summary

| Term                | Meaning                                                                      |
| ------------------- | ---------------------------------------------------------------------------- |
| Queue Depth         | Maximum number of commands a Submission Queue can contain                    |
| I/O Depth (iodepth) | Number of currently outstanding I/O commands                                 |
| Fixed or Dynamic?   | Queue depth is fixed when queue is created; iodepth changes during operation |
| Relationship        | iodepth ≤ queue depth                                                        |
| fio iodepth         | Number of outstanding I/Os maintained by the workload generator              |
| Performance Impact  | Higher iodepth generally increases NVMe throughput and IOPS until saturation |

A common NVMe benchmark configuration is:

```
Queue Depth = 64fio iodepth = 32
```

meaning the queue can hold 64 commands, but the workload intentionally keeps only 32 commands outstanding at any given time.

