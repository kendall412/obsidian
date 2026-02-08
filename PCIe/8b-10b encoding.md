8b/10b encoding is a line code that maps 8-bit data bytes into 10-bit symbols, crucial for high-speed serial communication standards (like Fibre Channel, PCIe) by ensuring DC balance (equal 1s and 0s on average) and sufficient bit transitions for reliable clock recovery (CDR) at the receiver, preventing signal drift and aiding synchronization, despite its 20% bandwidth overhead. It achieves this by using specific 10-bit codes, including special control (K) characters, chosen based on the running disparity (difference between 1s and 0s) to keep the overall stream balanced. 

How it works

Splits data: An 8-bit byte is divided into a 5-bit and a 3-bit group.

Maps to codes: These groups are mapped to 6-bit and 4-bit codes, respectively, from pre-defined tables.

Maintains disparity: The selection of codes (e.g., from two possible 10-bit codes for some data) depends on the running disparity, ensuring the total number of 1s and 0s stays balanced over time.

Adds overhead: The 2 extra bits per byte add a 25% overhead, meaning a 1 Gbps data stream needs a 1.25 Gbps physical link.

Includes special characters: It uses special K-characters (e.g., commas like 0011111 and 1100000) for control signals, not just data. 

Key Benefits

DC Balance: Prevents DC offset, crucial for optical links and maintaining signal integrity.

Clock Recovery: Ensures enough 0-to-1 or 1-to-0 transitions for the receiver to lock onto the clock.

Error Detection: Can detect single-bit errors and other transmission faults. 

Common Uses

Fibre Channel

Gigabit Ethernet (1000BASE-X)

PCI Express (PCIe)

Serial Attached SCSI (SAS)

USB 3.0 (SuperSpeed USB)