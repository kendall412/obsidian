# Descriptors

Descriptors are data structures that provide metadata describing specific attributes, capabilities, or states of the NVMe subsystem. They act as a standardized "ID card" or "spec sheet" that the host (CPU/OS) reads to understand how to communicate with the drive.

Unlike older protocols (like SATA/AHCI) which often relied on fixed register bits, NVMe uses flexible, extensible descriptor structures.

Here are the primary types of descriptors in NVMe:

1. Identify Data Structures (The Most Common "Descriptors")

    When a host sends the Identify command, the controller returns large data structures often referred to as descriptors. These define the drive's capabilities.

    - Controller Data Structure: Describes the controller's version, supported features, error recovery settings, and maximum queue sizes.
    - Namespace Data Structure: Describes a specific namespace (logical drive), including its size, block size (LBA format), and performance characteristics.
    - Namespace List: A list of all active namespace IDs.
    - Extended Data: Newer NVMe versions (1.3+) include descriptors for specific features like Zoned Namespaces (ZNS), Endurance Groups, and NVM Sets.

2. Error Information Descriptors

    When the host requests the Error Information Log, the controller returns a list of error entries. Each entry is a descriptor containing:

    - Error Count: How many times this specific error occurred.
    - LBA (Logical Block Address): Where on the disk the error happened.
    - Status Field: The specific NVMe status code (e.g., "Write Fault," "Unrecovered Read Error").
    - Command Specific Info: Details about the command that failed.
    - Time Stamp: When the error occurred.

3. Self-Monitoring, Analysis, and Reporting Technology (SMART) / Health Descriptors

    Part of the SMART/Health Information Log, these descriptors provide real-time health data:

    - Temperature: Current and critical thresholds.
    - Available Spare: Percentage of spare capacity remaining.
    - Percentage Used: Wear level indicator.
    - Data Units Read/Written: Total throughput lifetime stats.
    - Power Cycles & Hours: Usage duration.

4. Reservation Descriptors

    Used in multi-host environments (where multiple servers access the same storage array):

    - These descriptors track which host currently holds a reservation on a namespace.
    - They prevent data corruption by ensuring only one host writes to a specific area at a time.
    - They include the Host ID (a unique 64-bit identifier) and the reservation type (e.g., Write Exclusive, Read Exclusive).

5. Firmware Slot Descriptors

    When checking firmware updates, the controller provides descriptors detailing:

    - Which firmware slot is active.
    - Which slot is selected for the next reset.
    - The revision number of the firmware in each slot.

6. Transport-Specific Descriptors (Fabrics)

    In NVMe over Fabrics (NVMe-oF) (using RDMA, TCP, or FC), additional descriptors are used during the connection establishment:

    - Discovery Log Page Entries: Describe available subsystems on the network (Target NQN, Transport Type, Address, Port).
    - Controller Properties: Describe queue configurations specific to the fabric transport.

## Why Are Descriptors Important?

1. Interoperability: They allow any OS (Linux, Windows, VMware) to talk to any NVMe drive without needing custom drivers for every model. The OS simply reads the descriptors to know what the drive can do.
2. Extensibility: As NVMe evolves (e.g., adding ZNS or Key-Value support), new descriptor fields can be added without breaking backward compatibility.
3. Management: System administrators use tools (like `nvme-cli` in Linux) to read these descriptors to monitor health, troubleshoot errors, and configure the drive.

### Example: Reading a Descriptor

If you run the command `nvme id-ctrl /dev/nvme0` in Linux, the output you see is the parsed Controller Data Structure descriptor. It tells you the Model Number, Serial Number, Firmware Version, and IEEE OUI Identifier.

In summary, NVMe descriptors are the standardized data blocks that define what the drive is, how healthy it is, what errors it has seen, and how it should be managed.

