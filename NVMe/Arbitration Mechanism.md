
> In NVMe, the **arbitration mechanism** is the logic inside the NVMe controller that decides **which Submission Queue (SQ) gets serviced next** when multiple queues contain pending commands. Because NVMe supports up to **64K Submission Queues**, many CPU cores, applications, or virtual machines can submit commands simultaneously. The controller needs a fair way to determine which queue's command to execute first. This is called **arbitration**.

# Why Arbitration Is Needed

Consider:

- SQ1 contains 1000 I/O commands
- SQ2 contains 10 I/O commands
- SQ3 contains 50 I/O commands

The controller cannot process all queues simultaneously forever because internal resources are limited:

- NAND channels
- DRAM
- DMA engines
- Command processing units

Therefore, the controller must decide:

> "Which queue should I service next?"

That decision process is arbitration.

# Where Arbitration Fits

```
Host
 │
 ├─ Submission Queue 1
 │
 ├─ Submission Queue 2
 │
 ├─ Submission Queue 3
 │
 ▼
 NVMe Controller
      │
      ▼
 Arbitration Logic
      │
      ▼
 Command Scheduler
      │
      ▼
 NAND/Flash Processing
```


# Arbitration Mechanisms Defined by NVMe

The NVMe specification defines two main arbitration methods:

1. Round Robin (RR)
2. Weighted Round Robin with Urgent Priority Class (WRRU)

The controller advertises support through the **Arbitration Mechanism Supported (AMS)** field in the CAP register.

# 1. Round Robin Arbitration

This is the simplest method.

The controller services queues one after another.

Example:

```
SQ1 → SQ2 → SQ3 → SQ1 → SQ2 → SQ3 ...
```

Suppose:

```
SQ1 : 100 cmds
SQ2 : 100 cmds
SQ3 : 100 cmds
```

Execution order:

```
Command from SQ1
Command from SQ2
Command from SQ3
Command from SQ1
Command from SQ2
Command from SQ3
...
```

Advantages:

- Simple
- Fair

Disadvantages:

- No prioritization

# 2. Weighted Round Robin with Urgent Priority (WRRU)

Allows higher-priority queues to receive more service.

The controller assigns queues to priority classes.

NVMe defines:

|Priority|Meaning|
|---|---|
|Urgent|Highest|
|High|High importance|
|Medium|Normal|
|Low|Background|

## Example

Weights:

```
Urgent = 8
High   = 4
Medium = 2
Low    = 1
```

Queues:

```
SQ1 = High
SQ2 = Medium
SQ3 = Low
```

Scheduler behavior:

```
SQ1 SQ1 SQ1 SQ1
SQ2 SQ2
SQ3
```

Then repeat.

Result:

```
High queue receives more bandwidth.
```

# Priority Classes

Each I/O Submission Queue has a priority field:

```
Create I/O Submission Queue Command
                 │
                 ▼
          Queue Priority
```

Possible values:

| Value | Priority |
| ----- | -------- |
| 00b   | Urgent   |
| 01b   | High     |
| 10b   | Medium   |
| 11b   | Low      |

The controller uses these priorities during arbitration.

# Arbitration Burst (AB)

NVMe allows the host to control how many commands are serviced from a queue before moving to another queue.

Configured through:

**CC.ARB**

(Controller Configuration Register)

Example:

```
AB = 4
```

Controller may execute:

```
SQ1 cmd1
SQ1 cmd2
SQ1 cmd3
SQ1 cmd4

then

SQ2 cmd1
SQ2 cmd2
SQ2 cmd3
SQ2 cmd4
```

before switching queues.

This reduces arbitration overhead.

# Arbitration Parameters

The **Arbitration Feature** (Feature Identifier 01h) allows the host to configure:

|Parameter|Purpose|
|---|---|
|HPW|High Priority Weight|
|MPW|Medium Priority Weight|
|LPW|Low Priority Weight|
|AB|Arbitration Burst|

These values affect WRRU scheduling.

# Arbitration Feature Example

Host sends:

```
Set Features
FID = 01h (Arbitration)
```

Parameters:

```
HPW = 8
MPW = 4
LPW = 1
AB  = 4
```

Meaning:

```
High Priority Queue:
  gets 8 shares

Medium Priority Queue:
  gets 4 shares

Low Priority Queue:
  gets 1 share
```

The controller will favor high-priority queues accordingly.

# Admin Queue Arbitration

The **Admin Submission Queue (ASQ)** is special.

Administrative commands include:

- Identify
- Get Log Page
- Create Queue
- Firmware Download
- Firmware Commit

Admin commands generally have higher importance and are processed independently from normal I/O queue arbitration.

```
Admin Queue
    │
    ├─ Firmware Update
    ├─ Identify
    └─ Log Retrieval

I/O Queues
    │
    ├─ Read
    ├─ Write
    └─ Flush
```

The controller typically ensures admin commands are not starved by heavy I/O traffic.

# Real SSD Example

Imagine:

```
Queue 1:
Database Reads (High)

Queue 2:
User Writes (Medium)

Queue 3:
Background Garbage Collection (Low)
```

Without arbitration:

```
Database reads may be delayed.
```

With WRRU:

```
Database Reads  → serviced most often
User Writes     → serviced normally
Background Work → serviced occasionally
```

This improves latency for critical workloads.

# Arbitration vs Command Scheduling

These are different concepts.

### Arbitration

Chooses:

```
Which queue gets serviced next?
```

Example:

```
SQ1 or SQ2 or SQ3 ?
```

### Command Scheduler

Chooses:

```
How to execute the selected command?
```

Example:

```
Which NAND die?
Which channel?
Which DMA engine?
```

Flow:

```
Submission Queues
        │
        ▼
   Arbitration
        │
        ▼
 Command Scheduler
        │
        ▼
 NAND Execution
```

# Summary

> **NVMe arbitration** is the controller mechanism that selects which Submission Queue's command should be processed next when multiple queues contain pending commands.

Key points:

- Operates between Submission Queues and command execution.
- Ensures fairness and/or prioritization.
- Supports:
    - Round Robin (RR)
    - Weighted Round Robin with Urgent Priority (WRRU)
- Uses queue priorities:
    - Urgent
    - High
    - Medium
    - Low
- Controlled through:
    - CAP.AMS (supported arbitration methods)
    - Arbitration Feature (FID = 01h)
    - Arbitration Burst (AB)

In a heavily loaded SSD, arbitration is one of the key mechanisms that determines command latency, throughput fairness, and quality of service (QoS).


