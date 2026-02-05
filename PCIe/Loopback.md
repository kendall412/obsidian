In PCI Express (PCIe) link training sequences, loopback is a diagnostic testing mode where a device under test (DUT) is instructed to re-transmit the exact data it receives, effectively sending the data back to its source. This allows the transmitting device (the "Loopback Lead") to verify the integrity of the physical connection and the receiver's functionality without needing a fully operational link partner. 

Purpose and Function

Testing and Troubleshooting: Loopback is a critical mechanism for fault isolation and testing the physical layer (PHY) of a PCIe interface, especially during manufacturing (HVM testing) and compliance testing. It helps pinpoint errors in the design or physical connection.

Link Training without a Partner: It enables a device to perform link training and optimization (like equalization) even if the other end of the link isn't a typical, fully functional PCIe device (e.g., a tester or a simple board).

Verification: The transmitting device can compare the data it sent with the data it receives back to check for errors, signal integrity issues, or proper functioning of components like the serializer/deserializer and 8b/10b encoder/decoder. 

Mechanism

The PCIe specification includes a specific Link Training and Status State Machine (LTSSM) state dedicated to loopback. 

Initiation: A device initiates the loopback state by sending specific Training Sequence 1 (TS1) or TS2 ordered sets with a designated "Loopback bit" set in the training control field.

Follower Behavior: The receiving device, upon receiving these specific ordered sets, enters "Loopback Follower" mode. In this mode, it internally routes the received data stream back to its transmitter, without the data needing to go up to the higher layers of the PCIe protocol stack.

Data Flow: The data passes through key physical layer components (like the CDR, deserializer, 8b/10b decoder, and rate match FIFO on the receive side, and back through the rate match FIFO, 8b/10b encoder, and serializer on the transmit side).

Error Detection: The initiating device (Loopback Lead) monitors the looped-back data to ensure it matches the transmitted data, allowing for the detection of physical layer errors. 

This process is distinct from network-level IP loopback addresses (like 127.0.0.1) which are a software concept used for testing the local TCP/IP stack. The PCIe loopback is a physical layer electrical test and diagnostic feature.