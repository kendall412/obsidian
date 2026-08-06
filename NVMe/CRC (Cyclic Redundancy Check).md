
In PCIe, **CRC** means **Cyclic Redundancy Check**: an error-detecting code added to transmitted information so the receiver can detect corruption on the link.

PCIe mainly uses CRC in two places:

## 1. TLP CRC: LCRC and optional ECRC

A **TLP** is a **Transaction Layer Packet**, used for reads, writes, completions, messages, etc.

### LCRC — Link CRC

Every TLP normally has an **LCRC** appended by the **Data Link Layer**.

- It protects the TLP while it travels across a PCIe link.
- The receiver recalculates the CRC and compares it with the received LCRC.
- If the LCRC is wrong, the packet is considered corrupted.
- PCIe’s Data Link Layer can request/rely on replay of the TLP.

So LCRC is part of PCIe’s reliable link-level delivery mechanism.

### ECRC — End-to-End CRC

PCIe can also use an optional **ECRC**, usually generated/checked at the **Transaction Layer**.

- It protects the TLP end-to-end across switches and links.
- Unlike LCRC, which is checked and regenerated at each link hop, ECRC can remain with the packet from source to final destination.
- It is optional and used when stronger end-to-end error detection is needed.

## 2. DLLP CRC

A **DLLP** is a **Data Link Layer Packet**, used for link management, acknowledgments, flow control updates, etc.

DLLPs also include a CRC field to detect errors in those small control packets.

## Why PCIe uses CRC

CRC helps detect things like:

- Bit flips caused by electrical noise
- Signal integrity problems
- Transmission errors across a lane or link

If a CRC check fails, the receiver knows the packet was corrupted and can discard it or trigger PCIe’s error recovery mechanisms.

## Simple summary

In PCIe, **CRC is an error-detection value added to packets**.

- **LCRC** protects TLPs on each PCIe link.
- **ECRC** optionally protects TLPs end-to-end.
- **DLLP CRC** protects link-layer control packets.