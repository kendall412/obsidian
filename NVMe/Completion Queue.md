
A **Completion Queue (CQ)** is a **host-resident circular queue** used by the NVMe controller to **post the results (completions) of commands** that were submitted via a Submission Queue (SQ).

> If the SQ is where commands go _in_, the CQ is where results come _out_.

### Role in the NVMe queue pair model

NVMe operates with **queue pairs**:

- **Submission Queue (SQ)** → host → controller (commands)
- **Completion Queue (CQ)** → controller → host (completions)

Each SQ is typically associated with one CQ.

### What goes into a CQ

Each entry is a **Completion Queue Entry (CQE)**:

- Size: **16 bytes (4 DWORDs)**
- Written by the **controller** when a command completes

### CQE fields (high level)
```
DW0 : Command-Specific Result (e.g., DW0 result for admin cmds)
DW1 : Reserved (or command-specific)
DW2 : SQ Head Pointer (SQHD) | SQ Identifier (SQID)
DW3 : Command Identifier (CID) | Status Field (including Phase Tag)
```

Key fields:

- **CID**: matches the original command
- **SQHD**: tells host how far the SQ has been consumed
- **Status**: success/error + **[[Phase Tag (P)]]**
---
### How CQ works (step-by-step)

#### 1. Controller completes a command

- Executes command fetched from SQ

#### 2. Controller writes [[Completion Queue Entry (CQE)]]

- Places completion entry into CQ at **tail position**

#### 3. Controller updates CQ tail (internally)

- Advances its internal pointer

#### 4. Controller notifies host

- Via:
    - **Interrupt (MSI/MSI-X)**, or
    - Polling (host checks CQ)

#### 5. Host processes CQE

- Reads CQ entry
- Uses **CID** to match completion to command

#### 6. Host advances CQ head

- Updates its **head pointer**

#### 7. Host rings CQ doorbell

- Writes new head to **CQH doorbell**
- Signals:
    > “These entries are consumed”
---
### CQ structure (ring buffer)
```
CQ entries:

[0] [1] [2] ... [N-1]
 ↑               ↓
Head           Tail
```

- **Tail (controller-owned)** → where new completions are written
- **Head (host-owned)** → where completions are consumed

### Phase Tag (P bit) — critical detail

Each CQE includes a **Phase Tag**:
- Used to distinguish:
    - **New entries vs old (wrapped) entries**
- CQ is circular, so entries get reused

### Mechanism:

- Phase bit toggles each time the queue wraps
- Host tracks expected phase
---
### Types of Completion Queues

#### 1. [[Admin Completion Queue (ACQ)]]

- Queue ID = 0
- Used for admin commands

#### 2. [[I/O Completion Queues]]

- Queue ID ≥ 1
- Used for read/write completions
---
### Interrupt and affinity

- Each CQ can be mapped to:
    - A specific **MSI-X interrupt vector**
    - A specific CPU core

#### This enables:
- Per-core I/O processing
- Low latency and high scalability
---
### Key properties

- **Located in host memory (DRAM)**
- **Written by controller via DMA**
- **Lock-free ring buffer**
- Supports **deep queues (up to 64K entries)**

---
### Mental model

Think of CQ as:

> A **result mailbox** where the controller drops “done” notifications, and the host picks them up asynchronously.

### Key takeaway

> A **Completion Queue in NVMe** is a **host-memory circular queue where the controller posts command completion entries (CQEs)**, enabling asynchronous, high-performance I/O with minimal CPU overhead.


---
If you want, I can go deeper into:

- Exact **CQE bit-level layout (DW0–DW3)**
- Phase tag handling with a real example
- Or how CQ ties to **MSI-X interrupt routing and CPU affinity**










---


The completion queue (CQ) in NVMe stores entries that the controller writes after processing commands. Each entry in the CQ corresponds to a command submitted through a submission queue (SQ), and the host checks the CQ to track the status of these commands.

CQs are implemented as circular buffers in host memory. The host can either poll the CQ or use interrupts to be notified when new entries are available.

Completion Queue Element

Each completion queue element (CQE) in an NVMe CQ is an individual entry that contains status information about a completed command, including:

CID – Identifier of the completed command

SQID – ID of the SQ from which the command was issued

SQHD – Marks the point in the SQ up to which commands have been completed

SF – Indicates the status of the completed command

P – Flags whether the completion entry is new