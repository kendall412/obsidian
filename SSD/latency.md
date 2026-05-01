NVMe latency is the extremely low time delay (often under 20 microseconds) for data requests in NVMe storage, achieved by bypassing older protocols (like SATA/SAS) to use the direct PCIe bus, enabling massive parallel command queues for near-instant data access, crucial for demanding applications like AI, databases, and high-performance gaming by reducing bottlenecks.

**Latency in SSD performance testing** is the **time it takes to complete a single I/O operation**—from when the host issues a command until the completion is received.

# 🔹 What Exactly Is Measured

In NVMe terms, latency spans:

```
Host issues command → enters Submission Queue → controller processes → NAND access → Completion Queue entry → host receives completion
```

👉 So latency includes:

- Host stack overhead
- PCIe/NVMe command handling
- Controller processing
- NAND read/program/erase time


# Types of Latency (Important Distinction)

## 1. Read Latency

- Time to fetch data from NAND
- Typically **lowest latency**

**Typical NVMe SSD:**

- ~50–150 µs (microseconds)

---

## 2. Write Latency

- Includes:
    - NAND program time
    - possible buffering (SLC cache)
- Higher than read

**Typical:**

- ~100–500 µs (can spike higher under load)

---

## 3. Flush / Sync Latency

- Ensures data is persisted (no volatile cache)
- Can be **much higher** due to durability guarantees

---

# 🔹 Latency Components (Decomposition)

Latency is often broken into:

```
Total Latency = Submission Latency (slat)
              + Completion Latency (clat)
```

### ✔ Submission Latency (slat)

- Time from application → command reaches device
- Includes:
    - syscall
    - driver
    - queue insertion

### ✔ Completion Latency (clat)

- Time device takes to process the command
- Includes:
    - controller scheduling
    - NAND access
    - internal operations

### ✔ Total Latency (lat)

- slat + clat

---

# 🔹 Units

- Microseconds (µs) → standard for SSDs
- Nanoseconds (ns) → sometimes for ultra-low latency
- Milliseconds (ms) → indicates slow operation or spikes

---

# 🔹 Latency vs IOPS Relationship

```
IOPS ≈ 1 / latency   (at Queue Depth = 1)
```

👉 Example:

- 100 µs latency → ~10,000 IOPS (QD1)

---

# 🔹 Latency Distribution (Critical in Testing)

Latency is not a single number. You must look at percentiles:

|Metric|Meaning|
|---|---|
|Average|Mean latency|
|P50|Median|
|P90|90% of I/Os below this|
|P99|Tail latency (very important)|
|P99.99|Worst-case behavior|

👉 Tail latency is critical for:

- databases
- real-time systems

---

# 🔹 Factors That Affect Latency

## Device-side

- NAND type (SLC < TLC < QLC latency)
- Garbage collection
- Wear leveling
- Thermal throttling

## Workload

- Queue depth (higher QD → higher latency per I/O)
- Random vs sequential
- Read vs write mix

## System

- CPU scheduling
- Interrupts vs polling
- PCIe contention

---

# 🔹 Steady-State vs Fresh Drive

- Fresh SSD → artificially low latency
- Steady-state → realistic latency (after fill + GC)

👉 Proper testing always uses **steady-state latency**

---

# 🔹 Example (fio Output Interpretation)

Using fio:

```
clat (usec): avg=85, p99=120, p99.99=500
```

- Avg = 85 µs → typical performance
- P99 = 120 µs → good consistency
- P99.99 = 500 µs → occasional spikes

---

# 🔹 Key Insight

- **IOPS measures how many**
- **Throughput measures how much**
- **Latency measures how fast each operation completes**

---

# 🔹 One-Line Summary

**Latency in SSD performance testing is the time required to complete a single I/O operation, and it is the most fundamental metric driving IOPS and user-perceived responsiveness.**

---

If you want, I can go deeper into:

- **latency vs queue depth curves (very important for NVMe tuning)**
- or **how NAND program/read timings translate into observed latency**













---
### How NVMe Lowers Latency
- Direct PCIe Connection: NVMe drives connect directly to the CPU via the fast PCIe bus, avoiding the slower SATA/SAS interfaces and their protocol overhead.

- Parallelism: It supports up to 64,000 command queues (vs. SATA's one), allowing many operations to happen simultaneously, drastically reducing wait times.

- Efficient Protocol: The NVMe protocol itself is designed for low overhead, minimizing software and network delays, especially in networked storage (NVMe-oF).

### Latency Comparison (Typical)
- NVMe SSDs: Under 20 microseconds (µs).

- SATA SSDs: 50–150 µs.

- NVMe-oF (Network): Around 20–30 µs (approaching local NVMe speeds).

### Why It Matters
- Performance: Enables faster application loading, smoother gameplay, and quicker database responses.

- Scalability: Allows servers to support more virtual machines and demanding workloads without I/O bottlenecks.

- Consistency: Maintains steady, low latency even under heavy, simultaneous I/O operations, unlike older drives that slow down.