
`0x07` in the NVMe **Set Features** command is the **Number of Queues** feature. It is used by the host to request how many **I/O Submission Queues** and **I/O Completion Queues** the controller should allocate resources for.

This feature does **not** create the queues directly. It only negotiates the number of I/O queues that may later be created using:

- **Create I/O Completion Queue**
- **Create I/O Submission Queue**

### Feature ID

```text
FID = 0x07
Feature = Number of Queues
```

### What it controls

It controls the number of **I/O queues**, not the Admin queues.

The host provides requested values in Command Dword 11:

```text
CDW11[15:0]   = NSQR — Number of Submission Queues Requested
CDW11[31:16]  = NCQR — Number of Completion Queues Requested
```

Both fields are **zero-based**.

So:

| Field value | Meaning |
|---:|---:|
| `0` | 1 queue requested |
| `1` | 2 queues requested |
| `3` | 4 queues requested |
| `n` | `n + 1` queues requested |

Example:

```text
NSQR = 3
NCQR = 3
```

means the host requests:

```text
4 I/O Submission Queues
4 I/O Completion Queues
```

### Completion result

The controller returns the number of queues it actually allocated in Completion Dword 0:

```text
DW0[15:0]   = NSQA — Number of Submission Queues Allocated
DW0[31:16]  = NCQA — Number of Completion Queues Allocated
```

These are also **zero-based**.

So if completion DW0 returns:

```text
NSQA = 1
NCQA = 1
```

then the controller allocated:

```text
2 I/O Submission Queues
2 I/O Completion Queues
```

### Important behavior

- This feature is normally set during controller initialization.
- It should be set before creating I/O queues.
- It does not include the Admin Submission Queue or Admin Completion Queue.
- The controller may allocate fewer queues than requested.
- After this negotiation, the host creates queues up to the allocated limit.

### Example

If a host sends Set Features with:

```text
FID   = 0x07
CDW11 = 0x00030003
```

That means:

```text
NSQR = 0x0003 → request 4 I/O Submission Queues
NCQR = 0x0003 → request 4 I/O Completion Queues
```

If the completion returns:

```text
DW0 = 0x00010001
```

That means:

```text
NSQA = 0x0001 → allocated 2 I/O Submission Queues
NCQA = 0x0001 → allocated 2 I/O Completion Queues
```

So the host should only create up to 2 I/O submission queues and 2 I/O completion queues.