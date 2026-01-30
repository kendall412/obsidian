Controller Registers

![[Pasted image 20260129210623.png]]

NVMe controllers feature several memory-mapped registers located within memory regions defined by the Base Address Registers (BARs). These registers convey the controller's status, enable the host to configure operational settings, and facilitate error reporting.

  

Key NVMe controller registers include:

CC – Controller Configuration – Configures the controller's operational parameters, including enabling or disabling it and specifying I/O command sets

CSTS – Controller Status – Reports the controller's status, including its readiness and any fatal errors

CAP – Capabilities – Details the capabilities of the controller

VS – Version – Indicates the supported NVMe version

AQA – Admin Queue Attributes – Specifies the sizes of the admin queues

  

Initialization

Initializing an NVMe controller involves configuring the controller registers and preparing the system to communicate with the NVMe device. The process typically follows these steps:

1. The host clears the enable bit in the CC register (sets it to 0) to reset the controller.

2. The host configures the controller by setting initial parameters in the CC register and enables it by setting the enable bit.

3. The host sets up the admin submission and completion queues to handle administrative commands.

4. Administrative commands are issued to retrieve namespace information and perform other setup tasks.

5. The host creates I/O submission and completion queues, preparing the controller for I/O operations.

  

Reset and Shutdown

Reset and shutdown operations ensure the proper handling of the controller and the maintenance of data integrity. The reset process involves setting the enable bit in the CC register to 0, which stops all I/O operations and clears the controller's state, returning it to a known state.

  

Shutdown is initiated by setting the shutdown notification (SHN) field in the CC register. This allows the controller to halt operations, flush caches, and ensure that any data in flight is safely handled before powering off or resetting