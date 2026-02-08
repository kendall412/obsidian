To perform an SSD endurance test with `fio`, you write sustained, often random, large amounts of data to the drive for extended periods (days/weeks) using a configuration file (`.fio`) to simulate real-world or worst-case workloads (like 4K random writes with high queue depth), often preconditioning the drive first with sequential writes to fill it completely, then monitoring performance and drive health metrics (like [[Total Bytes Written (TBW)]]) to gauge endurance degradation. Key parameters involve `rw=randrw` or `randwrite`, large `iodepth`, `bs` (block size), and `runtime` or `size` set to fill the drive many times over, targeting the raw device (`/dev/nvme0n1`) for direct, unbuffered I/O.

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