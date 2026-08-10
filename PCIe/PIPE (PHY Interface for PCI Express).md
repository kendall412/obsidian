## PIPE (PHY Inteface for PCI Express)

When designing or debugging PCIe hardware (such as on FPGAs or ASICs), developers use the PIPE (PHY Interface for PCI Express) specification. PIPE defines the communication between the Media Access Control (MAC) layer and the physical transceiver (PHY).

These are commonly exposed as PHY power-down states in the PIPE interface.

| PIPE PHY state | Typical use |
|---|---|
| **P0** | Fully active PHY state; corresponds to PCIe **L0** operation. |
| **P0s** | Fast low-power PHY state; commonly used for **L0s**. |
| **P1** | Lower-power PHY state; often associated with **L1**. |
| **P2** | Deeper PHY power-down state; used for deeper idle/link-off conditions. |
| **P3** | Very low-power/off-like PHY state, depending on PIPE version and implementation. |
| **P4** | Present in newer PIPE versions; used for very deep low-power states such as L1 substates in some implementations. |

