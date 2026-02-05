In PCI Express (PCIe), a Hot Reset is an in-band, non-fundamental reset mechanism triggered across the active PCIe link itself, rather than via a dedicated hardware signal line. It is a software-initiated event used for various maintenance tasks, including link re-training, without requiring the system to power down or reboot. 

How it Works

The Hot Reset process is part of the Link Training and Status State Machine (LTSSM) and is communicated via the physical layer's ordered sets. 

Initiation: A component (typically the Root Port, often via software control) initiates a Hot Reset by sending Training Sequence 1 (TS1) ordered sets with a specific "Hot Reset" bit asserted over the existing link.

Detection: The receiving device (e.g., an endpoint) detects the incoming TS1 ordered sets with the Hot Reset bit set.

Reset State Entry: Both the transmitter and receiver sides of the link enter the Hot Reset state within the LTSSM. This puts the device in a reset condition but some elements, like power management logic, may remain unaffected, providing a "gentler" reset compared to a fundamental reset.

Link Re-training: After a brief timeout in the Hot Reset state, the link transitions back to the Detect state and subsequently goes through the full link training process (Detect -> Polling -> Configuration -> L0) to re-establish communication and negotiate link parameters. 

Key Characteristics

In-band: It uses the existing PCIe data lines for signaling, not a separate pin like the PERST# signal used for a fundamental reset.

Non-fundamental: It does not reset all hardware logic and configuration registers throughout the entire device as a fundamental reset (cold or warm) would.

Software-Triggerable: It can often be triggered by system software, providing greater flexibility for managing devices dynamically.

Use Cases: It is often used during hot-plug events (adding or removing a card without power cycling) or for error recovery when the link integrity needs to be restored.