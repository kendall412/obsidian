![[pcie_architecture_2.png]]

The root complex in PCI Express (PCIe) is the intermediary between the system’s central processing unit (CPU), memory, and the PCIe switch fabric that includes one or more PCIe or PCI devices. It uses the [[Link Training and Status State Machine (LTSSM)]] to manage connected PCIe devices. The LTSSM detects, polls, configures, recovers, resets, and disables the devices as required during operation.

The PCIe root processes transaction requests received from the CPU and connects to devices on the PCIe bus. A root complex often contains more than one PCIe port; multiple PCIe endpoints, bridges, and switches can connect to the ports on the root complex or be cascaded throughout the system

A master copy of a “Type 1 Configuration Table” is in the root complex and defines the host memory space accessible from each Endpoint device. Each Endpoint device holds the master copy of its own memory space in the host system as a “Type 0 Configuration Table.” Type 1 and Type 0 configuration tables are configured by the host operating system that controls the root complex. A PCIe bridge works as a tiered root complex with its own “Type 0 Configuration Table.” In addition to traditional applications, PCIe is being used as the main system interconnect technology in multi-host and embedded applications. 

## Key functions of the root complex

- Transaction bridging: It translates and bridges transactions between the CPU/memory subsystem and the PCIe bus. 

- Device discovery and configuration: During the boot process, it scans the PCIe bus to detect and identify all connected devices. 

- Resource management: It allocates system resources, such as memory and I/O addresses, to the devices on the bus.

- Communication initiation: As the master of the PCIe hierarchy, it initiates all transactions on behalf of the CPU. 

- Error and power management: It handles error reporting and manages the power states of the connected devices.