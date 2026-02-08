Input/Output Operations Per Second (IOPS) is a key performance metric for storage devices (HDDs, SSDs, SANs) <span style="color:yellow">measuring how many read/write operations they handle in one second</span>. High IOPS indicates faster data access and better performance, critical for database applications, virtualized environments, and cloud infrastructure. 

<p>Queue = IOPS * Latency</p>
<p>IOPS = <sup>Queue</sup>&frasl;<sub>Latency</sub></p>

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