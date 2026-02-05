
Lanes

The PCIe protocol communicates data through a set of serial ‘lanes’. Electrically, these are a pair of AC coupled differential wires. The number of lanes for an interface can be of differing widths, with x1, x2, x4, x8, x12, x16 and x32 supported. Obviously the higher the number of lanes the greater the data bandwidth that can be supported. Graphics cards, for instance, might be x16, whilst a serial interface might be x1. However, an interface need not be connected to another interface of the same size. During initialization, active lanes are detected, and a negotiated width is agreed (to the largest of the mutually supported widths). The interface will then operate as if both ends are of the negotiated width. The diagram below shows the motherboard PCIe connectors of my PC supporting 3 different lane widths: x16 at the top, x1 (two connectors), and x4.

![[pcie_slots.png]]

Note that the x4 connector at the bottom has an open end on the right. This allows a larger width card (e.g., x16) to plug into this slot and operate at a x4 lane configuration. The signals are arranged so that the lower lane signals are on the left (with respect to the above diagram). It is outside the scope of this article to go into physical and electrical details for PCIe as we are concentrating on the protocol but signals other than the lane data pairs include power and ground, hot plug detects, JTAG, reference clocks, an SMBus interface, a wake signal, and a reset.
