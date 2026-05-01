
### NVMe Hierarchy

An **NVM subsystem** in NVMe is basically the **entire storage device as a whole**, including everything that makes it work—not just a single controller.

## 🧠 Simple intuition

> **NVM subsystem = the full NVMe device (all controllers + all storage + shared logic)**

## 🔑 Key definition (clean version)

> An **NVM subsystem** is a collection of one or more NVMe controllers, namespaces, and shared resources that together provide non-volatile storage.


```
[NVM Subsystem]
    ├── Controller 1
    │      ├── Admin Queue
    │      └── I/O Queues
    │
    ├── Controller 2 (optional)
    │      ├── Admin Queue
    │      └── I/O Queues
    │
    ├── Namespaces (ns1, ns2, ...)
    │
    └── Shared Resources
           ├── NAND flash (actual storage)
           ├── FTL (address translation)
           ├── DRAM / cache
           └── Firmware state
```

## 🧱 What’s inside an NVM subsystem?

An NVM subsystem can include:

### 1. 🎛️ One or more controllers

- Each controller is like an interface the host talks to
- A device can have:
	1. Single [[Controller]] (typical consumer SSD)
    2. Multiple [[Controller]]s (enterprise, multi-port SSDs)

### 2. 💾 Namespaces (storage volumes)

- Logical storage units exposed to the host
- Like “disks” or partitions from the host perspective

### 3. ⚙️ Shared resources

These are **critical for understanding NSSR**:

- Flash media (NAND)
- DRAM/cache
- Flash Translation Layer (FTL)
- Firmware state
- Internal queues / schedulers
    
👉 These are **shared across controllers**, which is why subsystem-level issues happen.

## 🏗️ Example layouts

### 🟢 Consumer SSD

- 1 [[Controller]]
- 1 subsystem
- 1–few namespaces

👉 Controller ≈ subsystem (they’re basically the same)

### 🔵 Enterprise SSD (dual-port)

- 2 [[Controller]]s (for redundancy / multipath)
- 1 shared subsystem
- Shared NAND + FTL

👉 Controllers are independent interfaces, but **same underlying device**

## 🔄 Why the concept matters

Because different reset scopes exist:

- **Controller reset** → affects _one controller_
- **NVM subsystem reset (NSSR)** → affects _everything in the subsystem_







### NVM Subsystem Reset (NSSR)
NSSR in the NVMe protocol stands for NVM Subsystem Reset. It’s a reset mechanism that affects the entire NVMe subsystem, not just a single controller.

What that means:
- An NVMe subsystem can contain multiple controllers (especially in enterprise or multi-port devices).
- NSSR resets all of them at once, along with shared resources.

What happens during NSSR:
- All controllers in the subsystem are reset
- Outstanding (in-flight) commands are aborted
- Admin and I/O queues are cleared

The subsystem returns to a fresh initialization state (similar to power-on)

Why NSSR is used:
- Recover from serious or global errors
- Reinitialize shared hardware resources
- Ensure consistent state across multiple controllers

How it’s triggered:
- By writing to the NSSR register in the NVMe controller’s register space

Quick comparison:
- Controller Reset (CC.EN toggle) → resets one controller
- NSSR → resets the entire subsystem
- Shutdown (SHN) → graceful stop, not an abrupt reset

### NSSR is warranted when the problem is _subsystem-wide_, not controller-local

Here are the **real conditions** where issuing an **NVM Subsystem Reset (NSSR)** makes sense:

## 1. 🚨 Multiple controllers are failing

If you have a multi-controller NVMe device and see:

- Timeouts on **more than one controller**
- Admin commands failing across controllers
- Multiple controllers reporting **fatal status (CSTS.CFS)**
    

👉 Strong signal the issue is in **shared subsystem logic**, not one controller.

## 2. 🧠 Shared resource corruption

Things like:

- FTL (Flash Translation Layer) behaving inconsistently
- Metadata/cache corruption
- Internal firmware state machine stuck
    

Symptoms:

- Commands fail in strange/inconsistent ways
- Different controllers show similar weird behavior
    

👉 Resetting one controller won’t fix shared state → NSSR needed.

## 3. ❌ Controller reset didn’t recover the device

Typical escalation path:

1. Try controller reset (CC.EN toggle)
2. Recreate queues
3. Retry I/O
    

If:

- Controller comes back but still misbehaves
- Or immediately re-enters fatal state
    

👉 That’s when you escalate to NSSR.

## 4. 🧊 Device-wide hang / deadlock

- No forward progress on any queue
- Admin queue stops responding
- Doorbells don’t trigger completions
    

👉 Suggests firmware or hardware deadlock affecting the whole subsystem.

## 5. 🔁 Inconsistent state across controllers

Example:

- One controller thinks a namespace is valid, another doesn’t
- Different identify data across controllers
    

👉 Indicates broken synchronization → requires subsystem-wide reset.


## 6. ⚠️ Firmware-directed reset (rare but real)

Some devices internally decide:

- “State is unrecoverable”
- Signal host (or internally trigger equivalent behavior)
    

👉 NSSR used as recovery mechanism.


## 7. 🔌 After certain critical errors

Like:

- PCIe-level issues that don’t fully reset internal state
- Internal ECC or media errors cascading into firmware instability


## ❌ When NSSR is _NOT_ warranted

You _don’t_ use NSSR for:

- A single queue timeout
- One controller glitch
- Normal error recovery
- Performance hiccups
    

👉 That’s overkill and disrupts all I/O.

## 🧠 Mental model

Think of it like:

- **Controller reset** → “Restart this one process”
- **NSSR** → “Reboot the entire operating system”
    

## 🔑 Rule of thumb

> If the failure involves **shared state, multiple controllers, or survives a controller reset → NSSR is warranted**
