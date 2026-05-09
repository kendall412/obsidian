
> A **Submission Queue (SQ)** in NVMe is a **host-resident circular queue used to submit commands to the NVMe controller**.  It is a **ring buffer in system memory** where the host places NVMe commands (SQEs), which the controller fetches and executes.

### Architectural context

NVMe uses a **queue pair model**:

- **Submission Queue (SQ)** → host → controller (commands go _in_)
- **[[Completion Queue]] (CQ)** → controller → host (completions come _out_)

Each SQ is typically paired with a CQ.

### What goes into a Submission Queue

Each entry in the SQ is a **Submission Queue Entry (SQE)**:

- Size: **64 bytes**
- Contains:
    - Opcode (read, write, admin, etc.)
    - Namespace ID
    - Data pointers (**[[Physical Region Page (PRP)]] or [[Scatter-Gather List (SGL)]]**)
    - Command-specific fields

---
### How SQ works (step-by-step)

#### 1. Host prepares a command

- Builds a 64-byte SQE
- Fills:
    - Opcode (e.g., read/write)
    - Data buffer pointers
    - Parameters (LBA, length, etc.)

#### 2. Host writes SQE into SQ memory

- SQ resides in **host DRAM**
- Indexed by a **tail pointer**

#### 3. Host updates SQ Tail Doorbell

- Writes new tail value to a **doorbell register (MMIO)**
- This notifies the controller:
    
    > “New commands are available”
    
#### 4. Controller processes commands

- Fetches SQEs via DMA
- Executes commands asynchronously

#### 5. Completion is posted to CQ

- Controller writes completion entry into CQ
- Host later processes it

### Queue structure

#### Circular buffer (ring)
```
SQ entries:

[0] [1] [2] ... [N-1]
 ↑              ↓
Head           Tail
```

- **Tail (host-owned)** → where new commands are inserted
- **Head (controller-owned)** → where commands are consumed


### Key properties

#### 1. Host memory resident
- Located in **system DRAM**, not on the SSD

#### 2. Lock-free design
- No locks needed
- Uses head/tail pointers

#### 3. Multiple queues
- NVMe supports:
    - Up to **64K queues**
    - Up to **64K entries per queue**

This enables:
- Massive parallelism
- Per-core queue assignment
---
### Types of Submission Queues

#### 1. Admin Submission Queue (ASQ)
- Queue ID = 0
- Used for:
    - Identify
    - Create queues
    - Firmware commands

#### 2. I/O Submission Queues
- Queue ID ≥ 1
- Used for:
    - Read / Write
    - Flush
    - Dataset management

---
### [[Submission Queue Entry (SQE)]] overview

### Why SQ design matters

Compared to legacy interfaces (like AHCI):

- Eliminates register bottlenecks
- Enables **deep queues + parallel submission**
- Minimizes CPU overhead (doorbell batching, polling)


#### Key takeaway

> A **Submission Queue in NVMe** is a **host-managed circular command queue in memory that feeds commands to the controller via DMA**, forming the foundation of NVMe’s high-performance, parallel I/O model.

---
f you want, I can go further into:

- Exact **SQE field-by-field breakdown (DW0–DW15)**
- How **doorbell registers are mapped in PCIe BAR**
- Or **queue creation commands (Create I/O SQ)** and how queues are physically set up