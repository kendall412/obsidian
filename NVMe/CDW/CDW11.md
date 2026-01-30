Command DWORD (CDW) 11 in NVMe is a specific parameter that depends on the command being sent, often used for data set management (DSM) commands to override or provide flags and settings that would otherwise be determined by the command's flags. It can also be used in other commands to provide flags and the interrupt vector, as described in the OSDev Wiki. 

For Data Set Management (DSM) commands:

CDW 11 can be used to override any settings for attributes like discard or integral read/write that are specified by the command's flags.

For other commands:

CDW 11 can contain flags in its lower word and the interrupt vector in its higher word.

The interrupt vector is used with message-signaled interrupts (MSI/MSI-X) and should be the MSI vector plus 1, as MSI vector 0 is reserved for the admin completion queues.