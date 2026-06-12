
> A **queue pointer** in NVMe is an index that tracks the current position within a Submission Queue (SQ) or Completion Queue (CQ).

Since NVMe queues are implemented as **circular (ring) buffers**, both the host and the controller need pointers to know:

- Where new entries can be added.
- Where entries should be read from next.
- When the queue is full or empty.

# Submission Queue Pointers

For a Submission Queue, there are two important pointers:

```
Host owns:       SQ Tail (SQT)
Controller owns: SQ Head (SQH)
```

### SQ Tail (Host)

Points to the next free entry where the host will place a command.

### SQ Head (Controller)

Points to the next command the controller will process.

## Example

SQ depth = 8

```
Index

0    1    2    3    4    5    6    7
+----+----+----+----+----+----+----+----+
|Cmd0|Cmd1|Cmd2|    |    |    |    |    |
+----+----+----+----+----+----+----+----+
 ^
 |
SQH = 0

               ^
               |
SQT = 3
```

Meaning:

- Commands exist at entries 0,1,2.
- Next command will be written at entry 3.
- Controller will process entry 0 next.

# Completion Queue Pointers

For a Completion Queue:

```
Controller owns: CQ Tail (CQT)
Host owns:       CQ Head (CQH)
```

### CQ Tail (Controller)

Points to where the controller will place the next completion.

### CQ Head (Host)

Points to the next completion the host must process.

## Example

```
Index

0    1    2    3    4    5    6    7
+----+----+----+----+----+----+----+----+
|CQE |CQE |CQE |    |    |    |    |    |
+----+----+----+----+----+----+----+----+
 ^
 |
CQH = 0

               ^
               |
CQT = 3
```

Meaning:

- Completions exist at entries 0,1,2.
- Host has not processed them yet.
- Controller will write the next completion at entry 3.

# Why Are Pointers Needed?

Queues are ring buffers.

Suppose a queue has depth 8:

```
0 1 2 3 4 5 6 7
```

After reaching the end:

```
0 1 2 3 4 5 6 7
              ^
              Tail
```

the pointer wraps around:

```
0 1 2 3 4 5 6 7
^
Tail
```

This allows continuous reuse of queue entries.

# Submission Queue Example

Initially:

```
SQH = 0
SQT = 0
```

Queue empty.

Host submits three commands:

```
Write Cmd A
Write Cmd B
Write Cmd C
```

Now:

```
SQH = 0
SQT = 3
```

```
0    1    2    3
A    B    C
```

Controller processes Cmd A:

```
SQH = 1
SQT = 3
```

```
0    1    2
A    B    C
     ^
    SQH
```

Entry 0 can now be reused later.

# Completion Queue Example

Controller completes Cmd A:

```
CQH = 0
CQT = 1
```

Controller writes CQE at entry 0.

Host processes CQE:

```
CQH = 1
CQT = 1
```

Queue becomes empty again.

Host then updates:

```
CQ Head Doorbell = 1
```

to inform the controller.

# Relationship to Doorbells

The doorbell registers communicate queue pointers.

### Submission Queue

Host updates:

```
SQ Tail Pointer
```

and writes:

```
SQ Tail Doorbell
```

Example:

```
SQT = 15
```

Doorbell write:

```
SQ Tail Doorbell = 15
```

### Completion Queue

Host updates:

```
CQ Head Pointer
```

and writes:

```
CQ Head Doorbell = 10
```

telling the controller:

```
"I have consumed completions through entry 9."
```

# Complete Queue Ownership

```
Submission Queue

Host                    Controller

SQ Tail  ------------->  Reads
Writes Commands

SQ Head  <-------------  Updates
```

```
Completion Queue

Host                    Controller

CQ Head  ------------->  Reads
Consumes CQEs

CQ Tail  <-------------  Updates
Writes CQEs
```

# Visual Summary

```
Submission Queue

      Controller
          |
          v
+----+----+----+----+----+
|Cmd |Cmd |Cmd |    |    |
+----+----+----+----+----+
 ^              ^
 |              |
SQH            SQT

SQH = Next command to execute
SQT = Next free slot for host
```

```
Completion Queue

+----+----+----+----+----+
|CQE |CQE |CQE |    |    |
+----+----+----+----+----+
 ^              ^
 |              |
CQH            CQT

CQH = Next completion for host
CQT = Next free slot for controller
```

### Key Concept

A **queue pointer** is simply an index into a circular queue. NVMe uses:

- **SQ Head (SQH)** – controller's read pointer.
- **SQ Tail (SQT)** – host's write pointer.
- **CQ Head (CQH)** – host's read pointer.
- **CQ Tail (CQT)** – controller's write pointer.

> These pointers allow the host and controller to share queues efficiently without locks and support thousands of outstanding commands.

