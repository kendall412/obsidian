The PCIe addressing scheme uses a Bus, Device, and Function (BDF) triplet to uniquely identify every device and component in the system. This hierarchy is determined during the boot process through enumeration, where the Root Complex discovers devices and assigns them a unique address, like Bus 0, Device 1, Function 0. Devices then use this address to receive and transmit data packets, ensuring that all communication is routed correctly. 

Components of the addressing scheme

Bus: A group of devices and bridges. Multiple buses can exist in a complex system, typically starting with Bus 0 from the Root Complex. This identifier pinpoints the PCIe segment to which the device is tethered. Typically, on Intel motherboards, each PCIe slot correlates with a distinct segment. The allocation of these bus numbers is overseen by the motherboard chipset. 

Device: A specific component on a bus, such as a network card or a graphics card. ssigned to each PCIe device connected to a segment, this distinct number sets apart one device from another on the same bus. In Intel systems, this assignment is often managed by the system BIOS or firmware. 

Function: A specific logical component within a device. Given that PCIe devices can embody multiple functions—especially evident in PCIe cards featuring integrated USB controllers or network adapters—the function number serves to distinguish these various functions within a single device. For example, one device might have two functions, identified as Bus 0, Device 1, Function 0 and Bus 0, Device 1, Function 1. 

NVMe-Specific Functionality

The NVMe specification uses specific identifiers and concepts related to functionality and management: 

Namespaces: An NVMe drive can support multiple independent storage volumes called namespaces, each identified by a unique Namespace Identifier (NSID).

Submission/Completion Queues: Commands are managed through Submission Queues (SQ) and Completion Queues (CQ). The queue number is a specific identifier for each queue pair (Queue 0 is always the Admin queue).

Controller Functions: The specification also defines various commands (opcodes) and features that correspond to specific functions, such as Admin commands (e.g., namespace management, get log page) and NVM commands (e.g., read, write). 

In Linux environments, utilities like lspci are used to view the assigned PCIe bus, device, and function numbers. For example, a device might be listed as 3d:00.0, where 3d is the bus, 00 is the device number, and 0 is the function number.


How the addressing scheme is used

Enumeration: When a computer boots up, the system performs enumeration to discover all connected PCIe devices.

Address assignment: The system assigns each device a BDF address and allocates resources like memory and I/O space.

Data routing: When the CPU or another device needs to communicate with a specific endpoint, it sends a data packet addressed to that device's specific BDF triplet.

Unique identification: This ensures that data is routed to the correct component, even in a complex system with many devices and switches. 

This addressing protocol empowers the system to pinpoint each device on the motherboard, thereby facilitating efficient data transfer and communication among the CPU, chipset, and interconnected devices.

For instance, within a system boasting numerous PCIe devices:

A PCIe SATA controller housed in PCIe slot 1 might be designated as Bus 0, Device 1, Function 0.
Conversely, a PCIe Wi-Fi card situated in PCIe slot 2 could be denoted as Bus 0, Device 2, Function 0.

Furthermore, another PCIe SATA controller occupying the same segment as the first one may be identified as Bus 0, Device 1, Function 1.

This addressing paradigm streamlines the routing of data to and from each device, a process overseen by the system firmware and chipset to ensure the seamless operation of PCIe devices within the system.