In SSD testing, steady state is <span style="color: yellow">the long-term, stable performance level an SSD reaches after initial wear-in (FOB) and transition phases, characterized by consistent IOPS/throughput as NAND cells fill and background processes (like garbage collection) balance out write/erase cycles</span>. It reflects real-world, sustained performance, especially for data centers, where drives operate continuously, unlike client drives that idle, allowing TRIM/GC to refresh cells. Testing involves intense, prolonged random I/O to force the drive into this state, measuring performance without significant fluctuation over time. 

### Why Steady State Matters

- Real-World Performance: Data center SSDs operate 24/7, quickly reaching steady state, making this measurement crucial for enterprise applications.

- Consistent Behavior: It shows how an SSD performs under continuous heavy load, revealing its long-term reliability and sustained speed.

- NAND Management: It reveals the effectiveness of the drive's internal wear-leveling and garbage collection algorithms under stress. 

### How it's Tested

- Preconditioning: The drive is subjected to continuous, heavy writes (often random 4KB) to fill the NAND and force it out of its fresh state.

- Garbage Collection (GC): Background processes like garbage collection work to manage data, causing performance fluctuations.

- Reaching Stability: The drive eventually reaches a point where performance metrics (IOPS, throughput) don't change much over a measurement window, indicating steady state.

- Measurement: The final performance test is run in this stable condition, often after many hours, to get representative, sustained results. 

### Key Takeaway
While client SSDs benefit from idle time for TRIM/GC to maintain high "fresh" performance, enterprise SSDs are measured in steady state because they rarely idle, providing a truer picture of their long-term, sustained capability. 
