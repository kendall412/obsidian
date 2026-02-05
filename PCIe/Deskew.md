Lane-to-lane deskewing in PCIe is the process of synchronizing data arriving across multiple parallel lanes (like x4, x8, x16) by adjusting for tiny timing differences, ensuring all bits of a packet arrive aligned at the receiver, preventing data corruption, and maintaining high-speed link integrity, often using elastic buffers or skip ordered sets. Without it, data from different lanes could mix up, causing errors in multi-lane data streams. 

Why it's needed

Varying physical paths: Even on a single board, traces for different lanes have slightly different lengths and electrical properties, causing signals to arrive at different times.

Transceiver variations: Internal clock phases and buffers in the high-speed transceivers (SERDES) also introduce variations.

High speeds: At PCIe Gen5 (32 GT/s) and beyond, even picosecond-level skew matters, requiring precise alignment. 

How it works (in the receiver)

Elastic Buffers (FIFOs): The receiver uses elastic First-In, First-Out buffers in each lane, acting as variable-latency storage.

Skew Detection: The receiver detects the skew between lanes.

Latency Adjustment: It adjusts the latency of each lane's buffer (by adding or removing symbols) to align all data streams before presenting them to the higher layers (like the Data Link Layer).

Skip Ordered Sets (Skip OS): At the transmitter, special symbols (Skip OS) are sent at regular intervals, which the receiver detects to measure and correct skew. 

The goal

To guarantee that data from multiple lanes, which were sent together, is presented to the system's logic as if it arrived simultaneously, within very tight specifications (often less than one unit interval (UI) of skew).