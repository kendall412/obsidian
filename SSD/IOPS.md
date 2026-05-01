Input/Output Operations Per Second (IOPS) is a key performance metric for storage devices (HDDs, SSDs, SANs) <span style="color:yellow">measuring how many read/write operations they handle in one second</span>. High IOPS indicates faster data access and better performance, critical for database applications, virtualized environments, and cloud infrastructure. 

<p>Queue = IOPS * Latency</p>
<p>IOPS = <sup>Queue</sup>&frasl;<sub>Latency</sub></p>

**IOPS (Input/Output Operations Per Second)** is the primary metric for measuring how many **discrete read/write operations** an SSD can complete each second. It captures _transaction rate_, not data volume.

---

# 🔹 What IOPS Measures

- Count of completed I/O commands per second
- Each I/O = one NVMe command (read/write/flush, etc.)
- Independent of how much data each I/O transfers

👉 Think:

> IOPS = “how many requests per second the SSD can handle”


# 🔹 Relationship to Throughput

IOPS and throughput are linked via block size:

Throughput (MB/s) = (IOPS) (Block Size KB) / 1024

- Small blocks → high IOPS, lower throughput
- Large blocks → lower IOPS, higher throughput

**Example**

- 600K IOPS @ 4 KB → ≈ 2.34 GB/s
- 10K IOPS @ 128 KB → ≈ 1.25 GB/s

# 🔹 Typical IOPS Regimes (NVMe SSDs)

| Workload     | Block Size | IOPS (ballpark)           |
| ------------ | ---------- | ------------------------- |
| Random Read  | 4 KB       | 500K – 1.5M+              |
| Random Write | 4 KB       | 200K – 1M                 |
| Sequential   | 128 KB     | Low IOPS, high throughput |


# 🔹 What Determines IOPS

## 1) Latency (dominant factor)

IOPS ≈ 1/Latency

Lower latency → higher IOPS.

## 2) Queue Depth (QD)

- More outstanding commands → better parallelism
- NVMe scales IOPS with QD (up to controller limits)

## 3) Parallelism

- NAND channels, dies, planes
- NVMe queues mapped to CPU cores

## 4) Block Size

- Smaller blocks → more operations per second

## 5) Read vs Write

- Reads are faster (no program/erase)
- Writes involve NAND program + GC → lower IOPS


# 🔹 NVMe-Specific Advantages

Compared to SATA/AHCI:

- Many queues (up to 64K) × deep queues (64K entries)
- Lower software overhead
- MSI-X interrupts / polling

👉 Result: **orders-of-magnitude higher IOPS**

---

# 🔹 How IOPS Is Measured (Typical Test)

Using tools like fio:

- Workload: random read/write
- Block size: 4 KB (standard)
- Queue depth: varied (e.g., QD1 → QD128)
- Threads: 1 or multiple

**Example (fio-style):**

```
fio --name=randread --rw=randread --bs=4k --iodepth=32 --numjobs=4 --runtime=60
```



# 🔹 Common Pitfalls

- **Comparing IOPS without block size** → meaningless
- **Ignoring QD** → hides scaling behavior
- **Ignoring steady-state** → inflated fresh-drive numbers
- **Mixing read/write ratios** → inconsistent results

---

# 🔹 Quick Mental Model

```
IOPS = how many requests/sec
Throughput = how much data/sec
Bandwidth = max possible data/sec
```


# 🔹 One-Line Summary

**IOPS measures the number of I/O operations an SSD can complete per second, and it is primarily driven by latency, queue depth, and internal parallelism.**

---

If you want, I can:

- break down **IOPS vs latency curves (QD scaling)**
- or show **real NVMe fio output interpretation (lat, clat, slat, iops)**



---
### Key Aspects of IOPS

- Types: Random IOPS (scattered, high CPU usage) and Sequential IOPS (ordered, high throughput).

- Factors Affecting IOPS: Drive type (SSD > HDD), workload type (random vs. sequential), block size, and queue depth.

- Good Values: Modern SSDs commonly achieve thousands to over a million IOPS, while typical HDDs only manage 80–150.

- Calculation: Total I/O operations divided by the time in seconds.

- Maximum Size: In cloud environments like AWS, the maximum size of a single I/O operation is typically 256 KiB for SSDs and 1 MiB for HDDs. 

IOPS is essential for benchmarking speed, though other factors like throughput (bandwidth) and latency (speed of response) also influence overall system performance. 

To monitor IOPS with fio, run a command or job file with parameters like `bs=4k`, `rw=randread` (for reads) or `randwrite` (for writes) and observe the final output where fio reports the operations per second (e.g., `iops=47779`) after the test completes, often requiring `direct=1` and [[iodepth (fio)]] for accurate results, and can be enhanced with fio-parser for cleaner data.

Key `fio` Parameters for IOPS

- **`--filename=<device or file>`**: The target storage to test (e.g., `/dev/sda`, a specific path).
- **`--bs=4k`**: Block size, often 4KB, to measure operations rather than throughput.
- **`--rw=randread` / `randwrite`**: Specifies random read/write workload for IOPS testing.
- **`--direct=1`**: Bypasses the OS cache for more accurate hardware performance.
- **`--iodepth=<num>`**: Number of I/O requests in flight (e.g., 64, 256).
- **`--size=<size>`**: Total data size to process (e.g., 1G).
- **`--runtime=<seconds>`**: Duration of the test.
- **`--name=<jobname>`**: A name for the test.

## Random Reads
`sudo fio --filename=/dev/sdb --direct=1 --rw=randread --bs=4k --iodepth=128 --size=1G --runtime=60 --name=randread-iops-test`

## Mixed RW
`sudo fio --filename=/dev/sdb --direct=1 --rw=randrw --rwmixread=70 --bs=4k --iodepth=64 --size=1G --runtime=60 --name=mixed-iops-test`

## Interpreting the Results
````
read : iops=47779 (47779/47779), bw=186MiB/s(186/186), iops=47779, lat=2.67msec(2.67/2.67)

write: iops=15897 (15897/15897), bw=62.1MiB/s(62.1/62.1), iops=15897, lat=2.67msec(2.67/2.67)
````

`iops=47779`: This is the measured IOPS for reads in the example above.
`bw=186MiB/s`: Throughput (bandwidth) in MiB/s. 