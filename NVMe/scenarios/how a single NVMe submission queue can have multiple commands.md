
> A Submission Queue (SQ) is essentially an **array of command slots** in host memory. Each slot holds one 64-byte NVMe command. So when we say a Submission Queue contains multiple commands, we mean it contains multiple queue entries, where each entry is a separate NVMe command.

## Example: Submission Queue with Depth 8

Suppose the host creates an SQ with a depth of 8.

```
Submission Queue (SQ)

Entry 0 -> Read Command
Entry 1 -> Write Command
Entry 2 -> Flush Command
Entry 3 -> Read Command
Entry 4 -> Empty
Entry 5 -> Empty
Entry 6 -> Empty
Entry 7 -> Empty
```

Each entry is:

```
64 bytes
```

Therefore:

```
Queue Size = 8 × 64 = 512 bytes
```

## How Commands Are Added

The host maintains an **SQ Tail Pointer**.

Initially:

```
SQ Head = 0SQ Tail = 0
```

Queue is empty.

### Host submits first command

Host writes a Read command to Entry 0:

```
Entry 0 -> Read
```

Then increments Tail:

```
SQ Tail = 1
```

and rings the SQ doorbell.

Queue state:

```
Head = 0
Tail = 1

Entry 0 -> Read
```

### Host submits second command

Host writes a Write command to Entry 1:

```
Entry 0 -> Read
Entry 1 -> Write
```

Then:

```
SQ Tail = 2
```

Queue state:

```
Head = 0
Tail = 2
```

Now there are two outstanding commands.

### Host submits third command

```
Entry 0 -> Read
Entry 1 -> Write
Entry 2 -> Flush
```

Tail becomes:

```
SQ Tail = 3
```

Now three commands are waiting in the same SQ.

## How the Controller Sees Them

The controller maintains an **SQ Head Pointer**.

Suppose:

```
Head = 0
Tail = 3
```

The controller knows:

```
Commands available:
0
1
2
```

because:

```
Head != Tail
```

The controller fetches each command using DMA.

```
Fetch Entry 0
Fetch Entry 1
Fetch Entry 2
```

## Important: Commands Are Independent

The commands in a queue do not have to be identical.

Example:

```
Entry 0 -> Read LBA 100
Entry 1 -> Write LBA 200
Entry 2 -> Read LBA 500
Entry 3 -> Flush
Entry 4 -> Write LBA 700
```

One SQ can contain a mixture of commands.

## Outstanding Commands

A key NVMe advantage is that the host does **not wait** for one command to finish before submitting the next.

Instead:

```
Host:
  Submit Read
  Submit Write
  Submit Read
  Submit Flush

Controller:
  Process all of them
```

This is called having multiple **outstanding commands**.

Example:

```
SQ Depth = 64

Outstanding Commands = 50
```

meaning 50 commands are currently in the queue or being processed.

## Circular Buffer Behavior

Submission Queues are circular.

Example:

```
Depth = 8

Entry:
0 1 2 3 4 5 6 7
```

Suppose:

```
Head = 6
Tail = 2
```

Visually:

```
        Tail
          |
          v
+---+---+---+---+---+---+---+---+
|Cmd|Cmd|   |   |   |   |Cmd|Cmd|
+---+---+---+---+---+---+---+---+
 0   1   2   3   4   5   6   7
                      ^
                      |
                    Head
```

Valid commands are:

```
Entry 6
Entry 7
Entry 0
Entry 1
```

The queue wraps around.

## Real NVMe Read Example

Suppose an application reads 16 MB of data.

The NVMe driver may generate dozens or hundreds of Read commands.

```
SQ

Entry 0 -> Read LBAs 0-7
Entry 1 -> Read LBAs 8-15
Entry 2 -> Read LBAs 16-23
Entry 3 -> Read LBAs 24-31
...
```

All of these commands can sit in the same Submission Queue simultaneously.

The controller can then:

- Fetch them in order.
- Schedule NAND operations.
- Execute several in parallel across NAND channels and dies.

This is one of the main reasons NVMe achieves much higher performance than older interfaces such as ATA/SATA.

## Key Point

A Submission Queue is not a single command. It is a **circular array of command entries**:

```
Submission Queue
├── Command Entry 0
├── Command Entry 1
├── Command Entry 2
├── Command Entry 3
├── ...
└── Command Entry N
```

Each entry holds one 64-byte NVMe command. Therefore, a single Submission Queue can contain many commands at the same time, limited only by the queue depth (which can be up to 65,536 entries according to the NVMe architecture).

