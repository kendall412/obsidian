Throughput (measured in Mbps - Megabits per second) is <span style="color: yellow">the actual, real-world rate at which data is successfully transferred over a network, measures the volume of data transferred per second, crucial for sequential, high-bandwidth tasks like video editing and backups, representing the "useful" speed experienced by users</span>. Unlike theoretical bandwidth, throughput is affected by network congestion, packet loss, and hardware limits. 

of 1000 IOPS @ 4KB bs
throughput (TP) = IOPS * bs
               = (1000 IOPS)(4KB) = 4MB/s
          <p>IOPS=<sup>bs</sup>&frasl;<sub>TP</sub></p>
### Key Aspects of Throughput in Mbps:

- Real-World Speed: If a plan is 100 Mbps (bandwidth), actual throughput might be 70-90 Mbps due to overheads.

- Requirements: Basic browsing needs ~5-10 Mbps; HD streaming requires 15-25 Mbps, and 4K streaming demands 25+ Mbps.

- Measurement: Tools like iPerf3 or internet speed tests (e.g., Ookla) measure this, calculating the actual data transferred over time.

- Influencing Factors: High latency, low bandwidth, and network congestion reduce throughput. 

### Throughput vs. Bandwidth vs. Speed

- Bandwidth: The maximum theoretical capacity (the pipe size).

- Throughput: The actual data delivered (the amount of water flowing).

- Speed: Often used interchangeably with throughput to describe real-time performance. 