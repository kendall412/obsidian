To perform an SSD endurance test with `fio`, you write sustained, often random, large amounts of data to the drive for extended periods (days/weeks) using a configuration file (`.fio`) to simulate real-world or worst-case workloads (like 4K random writes with high queue depth), often preconditioning the drive first with sequential writes to fill it completely, then monitoring performance and drive health metrics (like [[Total Bytes Written (TBW)]]) to gauge endurance degradation. Key parameters involve `rw=randrw` or `randwrite`, large `iodepth`, `bs` (block size), and `runtime` or `size` set to fill the drive many times over, targeting the raw device (`/dev/nvme0n1`) for direct, unbuffered I/O.

> **Endurance testing** for an NVMe SSD is the process of **driving sustained write workloads over time** to verify how long the device can operate within spec before it **wears out or violates reliability/performance limits**.

---

## 🧠 What “endurance” means

> **Endurance = the total amount of data an SSD can reliably write over its lifetime.**

It’s governed by NAND physics (program/erase cycle limits) and managed by the controller’s FTL (wear leveling, garbage collection, over-provisioning).

## 📏 How endurance is specified

- **Terabytes Written (TBW)** — total bytes that can be written
- **[[Drive Writes Per Day (DWPD)]]** — writes per day over warranty
- **JEDEC client/enterprise workloads** (e.g., JESD218/219) — standardized profiles

Example:

- 1 DWPD on a 1 TB drive for 5 years ≈ **~1.8 PBW**

## 🔄 What an endurance test does

![https://images.openai.com/static-rsc-4/_wSdNw8i8qC08IePr9wE0B5G--YJ614fnRcStwStt3jepicpbNdR6Juihv8ejxvas5aw6bqlpfLYRaRan6owCtUTyNjk1PDwwD4ohNSA9BrdZkYEZk9JHiFbnZr8ftbajyPksrmydweoEuX5omo0RkB31C9CJurJ-tosypokqdfZQaXyspIL_mV0r5OTqooN?purpose=fullsize|375](https://images.openai.com/static-rsc-4/QjNH8_5vYUVUd0vFw4pnh2t6iFyVEaWEyY_LktDfoTA4w5CS4DYO-JHEil8MoTZyKDFTUfA-zartiS958JWrHD1_qgRM-1iaesLekQ6scMHStCyMs3IpVefGTIDTVZ5AT_7CgA2g9JMJQkI-npMbRPNc9ZZ6kspPTlLD1svSNjk?purpose=inline)

![https://images.openai.com/static-rsc-4/4msnAI9L6g8CHRp3NbciIGnsyxuEqG_2hzSC-Ey5jF9UzD3Nuu_BAZof0b-NPnTJlZPipLC2Px2xAE-6h6qY2kRM33rGmly7o1tvoImFZKk4GhuNSFm7qOWRvL3MrIXhaTgkTX-bwhEySue4UQZc75j36cPcp5XWZQ8-D9HXD0hGIUQV0JROKaifnROP1bJq?purpose=fullsize|392](https://images.openai.com/static-rsc-4/gl-M-lcYtkwFWC5U_sjMgPBRThS91TGHkOdyDsztvnMu7WWIWkjxZ9f-fZ40Kz9tz8bVzQQ7KOhBRuH5lcBksosVGvhHEGQE09OkQ6ReYVmtyTOKYbMi7SmZtre_FyjMgp0i6mFWAsey7Alg-qeZ6BlASsdm5OFDfoR2rZChTI4?purpose=inline)

![https://images.openai.com/static-rsc-4/H0dAy-lUs9aY9DejrNw3Dlc01N82qVV_u_boDXL8wW5WKiKDvzpsO6aPmP1PsZy1O9XrrCro5S7hs1RebIpZjaXNsbciRUQ7d59rngeBpJSYSJKV6ZoWOdRkt61Ib85G2iCfUgh0GC0n0VlUdiG84PW0vSQtNqMH5Z75snpRGdZtbp2xqhlnwakpgkyjAzDO?purpose=fullsize|564](https://images.openai.com/static-rsc-4/4FtJaIKkyeygokQQP_uHXvajuIyLePCaHgWOYvIhhqXq-ugH_ep8hFRkaKP2eebUfIkI-fPmQvvUgrYfgRdSpXdkRsOBDscfrW0IQJfc2jAKjp1-osuyOAavr_7fr-01VlK-EqC1r812CNUJm_5C8PAtH_ULEneHOc1E4KtzXso?purpose=inline)

### Core idea:

Continuously **write, verify, and monitor** the SSD until:

- It reaches a target (e.g., TBW spec), or
- It fails spec limits (performance, errors, SMART thresholds)

## ⚙️ Typical Test Methodology

### 1) Preconditioning

- Fill the drive (often 100%)
- Bring it to **steady state**

### 2) Sustained Write Workload

- Usually:
    - **Random writes (4K)** → worst-case wear
    - Or **JEDEC-defined mixes** (more realistic)
- Run continuously (days → months)

### 3) Periodic Measurements

- IOPS / throughput
- Latency (especially tail latency)
- Error rates

### 4) Data Integrity Checks

- Read-back verification
- Detect silent corruption (e.g., [[Cyclic Redundancy Check (CRC)]] mismatch)

## 📊 What You Monitor (Critical Metrics)

### 🔹 Wear Indicators

- **Percentage Used** (SMART)
- NAND **P/E cycle count**
- Wear-leveling count

### 🔹 Reliability

- Uncorrectable media errors
- ECC correction rates
- Bad block growth

### 🔹 Performance Drift

- Throughput degradation
- Latency increase (especially P99)

### 🔹 Write Amplification (WA)

- Internal writes vs host writes
- Higher WA → faster wear

## 🔬 Failure Criteria

An SSD is considered at/end-of-life when:

- **SMART “Percentage Used” ≈ 100%**
- Uncorrectable errors exceed spec
- Data retention fails
- Performance drops below guaranteed limits


## 🧪 Workload Types
|Workload|Purpose|
|---|---|
|4K Random Write|Worst-case stress (max wear)|
|Sequential Write|Lower stress, bandwidth-focused|
|Mixed R/W|Real-world simulation|
|JEDEC traces|Standardized client/enterprise behavior|

## 🔧 Tools

- **fio** (custom endurance loops)
- **nvme-cli** (SMART/log monitoring)
- Vendor validation tools (enterprise SSD qualification)

## ⚖️ Endurance vs [[performance]] Testing
|Aspect|Performance Testing|Endurance Testing|
|---|---|---|
|Goal|Speed (IOPS/MB/s)|Lifetime & durability|
|Duration|Minutes–hours|Days–months|
|Focus|Throughput, latency|Wear, errors, retention|
|State|Often steady state|Full lifecycle|

## ⚠️ Key Challenges

- Extremely **time-consuming**
- Requires **thermal control** (heat accelerates wear)
- Needs **power-failure scenarios** for realism
- Must track **large datasets and logs continuously**

---

## 🧩 Key Insight

> **Endurance testing doesn’t ask “how fast is the SSD?”—it asks “how long can it sustain correct operation under continuous stress?”**


If you want, I can:

- Design a **full NVMe endurance test plan (JEDEC-style)**
- Show **how to compute DWPD ↔ TBW precisely**
- Explain **how FTL algorithms (wear leveling, GC) influence endurance quantitatively**













---
### Prerequisites & Setup

1. Install FIO: sudo apt install fio (Debian/Ubuntu) or equivalent for your OS.

2. Identify Target Device: Use `lsblk` or `nvme list` to find your SSD (e.g., `/dev/nvme0n1`).

3. Secure Erase/Precondition: Secure erase the drive, then fill it with data (e.g., twice its capacity) using dd or fio to reach a steady state, as this reveals true endurance.

4. Create FIO Job File: Use a text editor to create a .fio file (e.g., endurance.fio)

```
# Example fio job file (endurance.fio)
[global]
ioengine=libaio
direct=1
buffered=0
group_reporting=1
time_based=1 # Run for a set time
ramp_time=60 # Warm-up time in seconds
runtime=3600 # Test duration in seconds (e.g., 1 hour for a quick test)

[ssd-write-stress]
filename=/dev/nvme0n1 # Target the raw device!
rw=randwrite # Or randrw for mixed
bs=4k # Small block size for intense NAND writes
iodepth=128 # High queue depth
numjobs=1 # Number of parallel jobs
size=100% # Write until full (or use a large fixed size like 100G)
```

### Running the Test

- Run as Root: sudo fio endurance.fio.

- Monitor: Observe performance (IOPS, throughput, latency) and check SMART data (`smartctl -a /dev/nvme0n1`) during and after the run.

- Repeat: Run the test for days/weeks, increasing runtime or size, and track how performance degrades to determine endurance.


### Key Parameters Explained

- **`ioengine=libaio`**: Asynchronous I/O for high performance.
- **`direct=1`, `buffered=0`**: Bypasses OS cache for raw device testing.
- **`rw=randwrite` / `randrw`**: Simulates random writes or mixed read/write, crucial for SSD wear.
- **`bs=4k`**: Small block size targets NAND flash cells directly.
- **`iodepth=128`**: High queue depth pushes the drive's performance limits.
- **`size=100%`**: Fills the entire drive.
- **`runtime=...`**: Specifies test duration.