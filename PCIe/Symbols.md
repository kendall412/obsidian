In PCIe, symbols are the fundamental units of data and control signaling, representing encoded groups of bits (like 10 bits in older Gen, 130 bits in newer Gen) that carry actual data bytes or special control functions (like starting a packet or linking) across the physical lanes, forming parts of Ordered Sets (OS) for link training or framing TLPs/DLLPs. They are crucial for the physical layer's encoding/decoding, ensuring data integrity and managing the link's state, with different symbols (e.g., K-symbols, COM, TS0, EIEOS) defining operations like data transmission, error correction, or link initialization. 

![[Pasted image 20260204175245.png]]

How Symbols Work in PCIe

Encoding: Raw data bytes are encoded (e.g., 8b/10b or 128b/130b) into symbols, adding control bits to differentiate data from control signals.

  

Types of Symbols:

Data Symbols: Represent actual data being sent.

Control Symbols (K-Symbols): Special codes used for framing, synchronization, and control, like STP (Start of TLP), END (End of TLP), SDP (Start of DLLP).

Ordered Set Symbols: Blocks of symbols (e.g., TS1, TS2) used during link training for clock alignment, state management, and initialization.

  

Framing Packets:

TLPs (Transaction Layer Packets): Start with an STP symbol and end with END/EDB symbols.

DLLPs (Data Link Layer Packets): Start with an SDP symbol and end with END symbols.

Role in Link Training: During initialization, specific Ordered Sets, made of symbols, help devices align clocks, detect lanes, and establish a stable, high-speed connection.

PCIe 6.0: Uses PAM4 signaling with 16-symbol Ordered Sets, where symbols are 8 bytes, allowing for high-speed operation. 

In essence, symbols are the building blocks that transform raw data into structured, manageable, and reliable signals over PCIe lanes, handling everything from packet framing to link synchronization.