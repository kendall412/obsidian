
NVM I/O Command Set
Used for actual data operations (the core workload). These are the commands that drive real storage performance.

Key characteristics:
Sent via I/O Submission Queues
High-performance, parallel execution
Common I/O Commands:
Read → Fetch data from SSD
Write → Store data
Flush → Ensure data is persisted
Write Zeroes → Efficiently zero blocks
Dataset Management (Trim/Deallocate) → Inform SSD of unused blocks