
A **race condition** occurs when the correctness of a system depends on the **timing or order** of multiple operations that are executing concurrently.

In other words:

> Two or more entities (threads, CPUs, processes, devices, or hosts) access the same data at the same time, and the final result depends on who "wins the race."

# Simple Real-World Example

Imagine a bank account containing:

```
Balance = $100
```

Two ATM transactions occur simultaneously:

### Transaction A

Withdraw $80.

### Transaction B

Withdraw $50.

Both ATMs read:

```
Balance = $100
```

at nearly the same time.

Then:

```
ATM A writes $20ATM B writes $50
```

Final balance:

```
$50
```

or

```
$20
```

depending on which write happens last.

Both results are wrong.

The correct balance should be:

```
$100 - $80 - $50 = -$30
```

This is a race condition.

# Computing Example

Suppose two CPU threads share a variable (C):

```
counter = 0;
```

Both execute:

```
counter = counter + 1;
```

simultaneously.

## What Actually Happens

Increment is not a single operation.

It is typically:

```
1. Read counter
2. Add 1
3. Write counter
```

### Thread A

```
Read counter = 0
```

### Thread B

```
Read counter = 0
```

### Thread A

```
Write counter = 1
```

### Thread B

```
Write counter = 1
```

Final value:

```
counter = 1
```

Expected:

```
counter = 2
```

One increment was lost.

# Race Condition in NVMe

Consider LBA 100 containing:

```
Value = X
```

Two hosts want to update it.

## Host A

```
Read X
Modify
Write Y
```

## Host B

At the same time:

```
Read X
Modify
Write Z
```

Possible execution:

```
Host A reads X
Host B reads X
Host A writes Y
Host B writes Z
```

Final value:

```
Z
```

Host A's update disappears.

This is called a **lost update**, a common race condition.

# Why Race Conditions Are Dangerous

They often cause:

- Corrupted data
- Incorrect calculations
- Filesystem corruption
- Database inconsistencies
- Random intermittent bugs

The worst part is that they may happen only occasionally.

Example:

```
Run #1 → Works
Run #2 → Works
Run #3 → Fails
Run #4 → Works
```

making them difficult to debug.

# How Race Conditions Are Prevented

Systems use synchronization mechanisms such as:
### Locks (Mutexes)

```
Thread A acquires lock
Thread B waits
```

Only one thread accesses the data at a time.

### Atomic Operations

CPU instruction example:

```
Compare-And-Swap (CAS)
```

The entire update occurs as one indivisible operation.

### Transactions

Databases use transactions so multiple updates either:

```
All succeed
```

or

```
All fail
```

### NVMe Fused Commands

NVMe uses fused operations to avoid races during:

```
Compare
+
Write
```

The controller guarantees no other command can interfere between the two operations.

# Visual Timeline

### Without Protection

```
Time →

Host A: Read X -------- Write Y
Host B:      Read X -------- Write Z

Final value = Z
```

Race condition occurs.

### With Atomic Protection

```
Time →

Host A: Compare+Write (atomic)
Host B: waits

Final value is deterministic
```

No race condition.

# Formal Definition

A race condition is:

> A situation in which multiple concurrent operations access shared state, and the system's behavior depends on the unpredictable order or timing of those operations.

In the context of NVMe, fused commands were introduced specifically to prevent race conditions such as **lost updates** when multiple hosts or threads attempt to modify the same logical blocks concurrently.