
Unlike its predecessor, PCI, PCIe is not a bus. It is a point-to-point protocol, more like AXI for example. The structure of the PCIe system consists of a number of point-to-point interfaces, with multiple peripherals and modules connected through an infrastructure, or fabric.

![[pcie_architecture.png]]

Unlike some other point-to-point architectures, there is a definite directional hierarchy with PCIe. The main CPU (or processor sub-system) sits at the top and is connected to a [[Root Complex]] using whatever appropriate user interface. This root complex is the top level PCIe interconnect component and would typically be connected to main memory through which the CPU system would access it. The root complex will have a number of PCIe interfaces included, but to a limited degree. To expand the number of supported peripherals ‘switches’ may be attached to a root complex PCIe interface to expand the number of connections. Indeed, a switch may have one or more of its interfaces connected to other switches to allow even more expansion. Eventually an ‘endpoint’ (EP) is connected to an interface, which would be on a peripheral device, such as a graphics card or ethernet network card etc.

At each link, then, there is a definite ‘downstream’ link (from an upstream component e.g., RC to a switch or EP) and an ‘upstream’ link (from a downstream component e.g., EP to switch/RC). For each link the specification defines three layers built on top of each other

- Physical Layer
- Data Link Layer
- Transaction Layer

The physical layer is concerned with the electrical connections, the serialization, encoding of bytes, the link initialization and training and moving between power states. The data link layer sits on top of the physical layer and is involved in data flow control, ACK and NAK replies for transactions and power management. The transaction layer sits on top of the data link layer and is involved with sending data packet reads and writes for memory or I/O and returning read completions. The transaction layer also has a configuration space—a set of control and status registers separate to the main address map—and the transaction layer protocol has read and write packets to access this space.

[[Physical Layer]]
[[Scrambling]]
[[Serial Encoding]]
[[Ordered Sets]]
[[Link Training and Status State Machine (LTSSM)]]
[[Compliance]]
[[SERDES Interface and PIPE]]

CONCLUSION
In this article we have looked at how PCIe is organized, with Root Complex, Switches and Endpoints, in a definite flow from upstream to downstream. We have seen that a PCIe link can be from 1 to 32 differential serial ‘lanes’. Bytes are scrambled (if data) and then encoded into DC free symbols (8b/10b or 128b/130b). Ordered Sets are defined for waking up a link from idle, link bit and symbol lock and lane-to-lane deskew. Training sequence Ordered sets are used to bring up a link from electrically idle to configured and initialized, configuring parameters as it does so or, optionally, forcing to non-standard states. Additional states are used for powered down modes of varying degrees, and a recovery state to update to higher link speed if supported. We also looked at the complementary PIPE specification for virtualizing away SERDES and PHY details to a standard interface.

We dwelt on the LTSSM at some length as this is the more complex aspect of the physical layer protocol, and the only remaining aspects of this layer are how the physical layer carries the higher data link layer and transaction layer packets.

**[https://www.linkedin.com/pulse/pci-express-primer-1-overview-physical-layer-simon-southwell/](https://www.linkedin.com/pulse/pci-express-primer-1-overview-physical-layer-simon-southwell/)**