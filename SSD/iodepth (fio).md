`iodepth` in NVMe refers to <span style="color: yellow">the number of I/O requests the operating system or benchmarking tool (like fio) issues to the drive concurrently, controlling command</span> [[queue depth]]  <span style="color: yellow">to maximize NVMe's parallel processing power, crucial for achieving high IOPS and throughput by filling the drive's internal queues for low latency</span>. Higher `iodepth` values (e.g., 32, 128, 256) increase parallelism, revealing the drive's true performance potential, while lower values (like `iodepth=1`) test worst-case latency, simulating single-threaded apps. 

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

