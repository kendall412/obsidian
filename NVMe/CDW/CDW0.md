CDW 0 (4.1.1 Admin Command and I/O Command Common SQE):

In NVMe, Command Dword 0 (CDW0) is the first 4-byte field in a 64-byte Submission Queue Entry (SQE). It contains essential parameters common to all Admin and NVM commands, specifically the Opcode and the Command Identifier (CID). 

![[Pasted image 20260129210234.png]]

![[Pasted image 20260129210247.png]]

![[Pasted image 20260129210256.png]]

"Command Dword 0, Namespace Identifier, Metadata Pointer, PRP Entry 1, PRP Entry 2, SGL Entry 1, and Metadata SGL Segment Pointer have common definitions for all Admin commands and I/O commands for all I/O Command Sets. Metadata Pointer, PRP Entry 1, PRP Entry 2, and Metadata SGL Segment Pointer are not used by all commands."

  

The structure of Command Dword 0 is as follows:

Opcode: An 8-bit field that specifies the type of command to be performed (e.g., Read, Write, Identify, Format NVM).

  

Command Identifier (CID): A 16-bit field assigned by the host software to uniquely identify a command within a specific Submission Queue. The combination of the CID and the Submission Queue ID (SQID) allows the controller and host to match a command with its corresponding completion entry.

The remaining bits in the 32-bit Dword are reserved for other flags or specific command details as defined in the NVMe base specification. 

  

CDW0 is a mandatory component of every NVMe command submission and is used by the controller to determine the action requested by the host. Other Dwords in the SQE (CDW10 to CDW15) may be used for command-specific parameters depending on the Opcode specified in CDW0.