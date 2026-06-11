`iodepth` in NVMe refers to <span style="color: yellow">the number of I/O requests the operating system or benchmarking tool (like fio) issues to the drive concurrently, controlling command</span> [[queue depth - iodepth]]  <span style="color: yellow">to maximize NVMe's parallel processing power, crucial for achieving high IOPS and throughput by filling the drive's internal queues for low latency</span>. Higher `iodepth` values (e.g., 32, 128, 256) increase parallelism, revealing the drive's true performance potential, while lower values (like `iodepth=1`) test worst-case latency, simulating single-threaded apps. 

### How it works
- `Queue Depth`: NVMe drives have multiple internal queues (Submission & Completion Queues) for commands, allowing them to handle many operations in parallel, unlike older HDDs.

- `iodepth` (FIO parameter): This setting tells fio how many commands to send into the device's queue before waiting for some to complete, essentially controlling the "concurrency" or "outstanding I/O".

- `numjobs`: Often used with `iodepth`, `numjobs` creates multiple parallel I/O streams, further stressing the drive and leveraging its multiple queues for higher aggregate performance. 

### Key Usage in NVMe Testing
- High IOPS/Throughput: Use high `iodepth` (e.g., 256) with multiple `numjobs` to saturate the NVMe drive and find maximum performance, as seen in benchmarks.

- Low Latency: `iodepth=1` (or `numjobs=1` with `iodepth=1`) tests the worst-case, single-command latency, useful for understanding application responsiveness.

- Real-World Simulation: Adjust `iodepth` and `numjobs` to mimic specific application behaviors, like databases needing high IOPS (high `iodepth`) versus simple sequential reads (lower `iodepth`).

### Examples
Test high concurrency (256 outstanding I/O operations)
`fio --name=nvme-test --iodepth=256 --rw=randread --bs=4k --size=1G --filename=/dev/nvme0n1 --direct=1 --ioengine=libaio`

 Test low latency (single I/O at a time)
`fio --name=nvme-test --iodepth=1 --rw=randread --bs=4k --size=1G --filename=/dev/nvme0n1 --direct=1 --ioengine=libaio`



# iodepth vs numjobs

In fio, **`numjobs`** and **`iodepth` (queue depth)** both increase concurrency, but they operate at **different layers of the I/O stack**. Confusing them leads to misleading NVMe results.

---

### 🔹 1. Definitions

#### ✔ `iodepth` (Queue Depth, QD)

- Number of **outstanding I/O requests per job**
- Directly maps to **NVMe Submission Queue occupancy**

```
1 job, iodepth=32 → up to 32 in-flight commands
```

👉 This is the **device-level parallelism driver**


#### ✔ `numjobs`

- Number of **independent threads/processes running the same workload**
- Each job has its **own file descriptor and I/O context**

```
numjobs=4 → 4 parallel workers
```

👉 This is **CPU / software-level parallelism**

### 🔹 2. Combined Effect

Total outstanding I/O:

```
Total QD ≈ numjobs × iodepth
```

### Example:

```
numjobs=4, iodepth=32 → ~128 outstanding I/Os
```

### 🔹 3. Key Difference

|Aspect|iodepth|numjobs|
|---|---|---|
|Scope|Per job|Across jobs|
|Layer|Device / queue level|CPU / process level|
|Maps to NVMe SQ|Directly|Indirectly|
|Overhead|Low|Higher (threads/processes)|

### 🔹 4. NVMe Interpretation

#### ✔ iodepth → fills submission queue

- Controls how deep the **NVMe SQ** is utilized
- Critical for achieving peak IOPS

#### ✔ numjobs → spreads load across cores

- Multiple jobs may:
    - Use same queue (shared)
    - Or multiple queues (depending on engine)


###🔹 5. Practical Scenarios

#### 🔸 Case 1: Single-thread scaling

```
numjobs=1, iodepth=1 → QD1 latency test
numjobs=1, iodepth=64 → deep queue test
```

👉 Measures **device capability per queue**

#### 🔸 Case 2: Multi-core scaling

```
numjobs=4, iodepth=1
```

👉 Simulates:

- 4 applications each doing synchronous I/O

#### 🔸 Case 3: High-performance NVMe test

```
numjobs=8, iodepth=32 → QD ≈ 256
```

👉 Matches:

- multi-core + deep queues
- real NVMe workload behavior

### 🔹 6. Important Nuances

#### ✔ Not perfectly linear

```
Total QD ≠ exact numjobs × iodepth
```

Because:

- OS scheduling
- I/O completion timing
- fio engine behavior


## ✔ Engine matters

With:

```
ioengine=libaio
```

- iodepth works as expected

With:

```
ioengine=sync
```

- iodepth ≈ 1 (ignored)

## ✔ NVMe multi-queue interaction

- Modern NVMe:
    - Multiple submission queues
    - Per-core mapping

👉 `numjobs` helps utilize **multiple hardware queues**

### 🔹 7. When to Use What

#### Use `iodepth` when:

- Studying **latency vs queue depth**
- Measuring **device saturation**
- Tuning NVMe queue behavior

#### Use `numjobs` when:

- Simulating **multiple applications**
- Testing **CPU/core scaling**
- Exercising **multi-queue NVMe**


### 🔹 8. Common Pitfall

```
numjobs=16, iodepth=1
```

≠

```
numjobs=1, iodepth=16
```

Why:

- First → 16 independent synchronous streams
- Second → 1 deep asynchronous stream

👉 They stress the SSD **very differently**

### 🔹 9. Mental Model

```
iodepth → depth of each pipe
numjobs → number of pipes
```

# 🔹 One-Line Summary

**`iodepth` controls how many I/Os are in flight per job (device-level parallelism), while `numjobs` controls how many independent jobs generate I/O (CPU-level parallelism); total load is roughly their product.**

---

If you want, I can:

- map this directly to **NVMe SQ/CQ and CPU core affinity**
- or analyze a **real fio output showing scaling with numjobs vs iodepth**