[https://wiki.osdev.org/NVMe](https://wiki.osdev.org/NVMe)

![[Pasted image 20260129210408.png]]

NVMe commands are structured within a 64-byte Submission Queue Entry (SQE), which is composed of 16 Double Words (DWORDs). Each DWORD is 4 bytes in size. The structure is generally consistent across different command types (Admin commands and I/O commands), with some fields having common definitions and others being command-specific.

  

Here's a general overview of the DWORD structure in an NVMe command:

  

Command DWORD 0 (CDW0): This DWORD is crucial as it contains the Opcode, which identifies the type of command being issued (e.g., Identify, Read, Write), and the Command Identifier (CID), used to uniquely identify the command.

  

Namespace Identifier (NSID): This 4-byte field specifies the namespace to which the command applies. If not applicable, it should be set to 0. A value of 0xFFFFFFFF can indicate that the command applies to all namespaces if supported by the controller.

  

Metadata Pointer (CDW4 and CDW5): These DWORDs, if used, point to the location of any associated metadata for the command.

  

Physical Region Pointers (PRPs) or Scatter-Gather List (SGL) Entry:

PRP Entry 1 and PRP Entry 2 (CDW6-9): These two 32-bit PRPs are used to point to the location of the data for the command in memory when using PRPs for data transfer.

SGL Entry 1: If the command uses SGLs for data transfer, this field is used instead of PRPs. 

  

Command-Specific DWORDs (CDW2, CDW3, CDW10-15): These DWORDs are reserved for command-specific parameters and vary depending on the particular NVMe command being executed. For example, a Store command in the Key Value Command Set might use CDW2, CDW3, CDW14, and CDW15 to specify parts of the KV key. 

  

Or, in the context of the NVMe Format NVM command (opcode 80h), Command DWORD 10 (CDW10) is used to specify formatting options. This DWORD might contain fields like the Protection Information (PI) settings, the size of metadata, or whether the command applies to a single namespace or all namespaces.

  

Another example is the Create I/O Submission Queue command, where Command DWORD 10 contains the queue identifier in the low word and the queue size in the high word. Command DWORD 11 holds flags in the low word and the interrupt vector in the high word. 

  

The NVMe Identify command uses specific CNS (Controller or Namespace Structure) codes within Command DWORD 10 to indicate the type of information requested, such as controller capabilities, namespace identification, or active namespace list.

  

In general, when constructing an NVMe command, the NVMe specification for that particular command details the purpose and bit-level layout of the relevant Command DWORDs (CDW0 to CDW15) within the Submission Queue Entry. These DWORDs are crucial for providing the necessary parameters and options for the controller to execute the command correctly.

  

Reserved Fields: Several DWORDs within the 16-DWORD SQE are reserved and should be initialized to 0 unless explicitly used by a specific command.

  

Completion Queue Entry (CQE): After a command is executed, a Completion Queue Entry is posted to the appropriate Completion Queue, providing status information about the command's execution. The CQE also contains DWORDs, including status values and command-specific information.

  

Key Aspects of Command Dwords

Structure: Each command submitted to the NVMe controller is a 64-byte structure in the Submission Queue. This structure is divided into various fields, many of which are designated as Dwords (4-byte chunks).

  

Common Fields: Several Dwords have common definitions across all commands, as defined in the NVM Express Base Specification:

  

Command Dword 0 (CDW0): This is a critical field that contains the Opcode (specifies the command, e.g., Read, Write, Identify, Abort) and the Command Identifier.

  

Namespace Identifier (NSID): Dword 1 typically holds the Namespace ID.

  

Data Pointers: Dwords are used to specify memory pointers (PRP entries or SGL entries) for data transfer between the host and the controller.

  

Command-Specific Fields: Other Dwords (typically CDW2, CDW3, and CDW10 through CDW15) are command-specific and used for parameters unique to that particular command.

  

For a Read or Write command, Dwords 10 through 14 are used for specifying the Starting Logical Block Address (LBA) and the number of logical blocks to transfer.

For an Identify command, the low byte of Dword 10 indicates the type of information to be returned (e.g., namespace, controller, etc.). 

  

In essence, Command Dwords are the instruction set parameters that allow the host software to tell the NVMe controller exactly what operation to perform, where the data is located, and where to put the results.

  
  

CDW 1:

In the NVMe command format, Command Dword 1 (CDW1) is typically the Namespace Identifier (NSID). This field specifies the particular namespace (a collection of logical blocks) that the command is intended to operate on. 

For most Admin and I/O commands, the NSID is a common, required field in the 

Submission Queue Entry (SQE) structure. 

  

Key details about Command Dword 1:

Purpose: It identifies the specific namespace the command targets.

  

Common Use Cases:

I/O Commands (Read, Write, etc.): The NSID is mandatory to indicate which part of the storage the operation should affect.

  

Admin Commands (e.g., Identify, Format NVM): It is used to specify the namespace to be identified or formatted. If the value is set to 0xFFFFFFFF, it may indicate that the command should apply to all namespaces, depending on controller support.

  

Command-Specific: While the NSID is the common definition, specific command sets (like the Key Value command set) or vendor-specific commands might use this Dword differently, but generally, the base specification's definition takes precedence. 

  

CDW 2:

In NVMe, Command Dword 2 (CDW2) is a command-specific field within the 64-byte Submission Queue Entry (SQE) structure. Its function and meaning are not universal across all NVMe commands; rather, the contents of CDW2 depend entirely on the specific command's opcode and the particular I/O command set being used (e.g., NVM Command Set, Key Value Command Set, Zoned Namespace Command Set). 

  

The DWORD (Double Word) term itself refers to an unsigned 32-bit (4-byte) unit of data. 

  

Here are examples of how Command Dword 2 is used in different contexts:

NVM Command Set (e.g., Read/Write commands): In standard NVM operations like Read and Write, Command Dwords 2 and 3 typically define the Starting 

  

Logical Block Identifier (SLI), specifying where on the storage medium the operation should begin.

  

Key Value (KV) Command Set: In the KV command set, Command Dwords 2 and 3 are used to contain specific bytes of the KV key (e.g., the first 8 bytes) for commands like Delete, Store, and Retrieve.

  

Generic Commands: For many generic or administrative commands, Command Dword 2 might be a reserved field, in which case it should be cleared to zero or ignored by the controller. 

  

Ultimately, the exact definition of Command Dword 2 for a specific command is detailed in the relevant NVM Express specification for the command set in use. 

NVM Express® Key Value Command Set Specification

May 18, 2021 — The Completion Queue Entry (CQE) structure and the fields that are common to all NVMe I/O Command Sets are defined in ...

  

CDW3:

Command DWORD 3 in NVMe is part of the command submission queue entry, but its function varies depending on the specific command. In the standard NVMe architecture, it is reserved. However, in the NVMe Key Value (KV) command set, it holds the lower 8 bits of the Key for certain commands like the Delete command. 

  

Command DWORD 3 function

Standard NVMe: In the general NVMe command structure, Command DWORD 3 is reserved for future use and is not assigned a specific function.

  

Key Value (KV) Command Set: For commands within the KV set, such as the Delete command, Command DWORD 3 is used to store the first part of the key.

It holds the lower 8 bits of the key (Key bytes [7:0]).

  

The remaining part of the key is stored in other command DWORDs, specifically Command DWORD 14 and 15. 

  

CDW4:

In the NVMe command structure, Command Dword 4 (CDW4) is a command-specific field used for different parameters depending on the particular command's function. It does not have a single, universal definition across all commands, unlike fields such as the opcode (in CDW0) or the Namespace Identifier (NSID). 

  

The Submission Queue Entry (SQE) for an NVMe command is a 64-byte data structure. The initial Dwords (0-3) contain common fields like Opcode, Command Identifier, and Namespace ID. The subsequent Dwords (Dword 4 through Dword 15) are command-specific fields, and their meaning is defined entirely by the operation code (opcode) of the command being issued. 

  

For example:

In a Read or Write command (NVM Command Set), CDW4 and CDW5 typically contain the starting Logical Block Address (SLBA).

  

For an Identify command (Admin Command Set), CDW4 is reserved, while other Dwords like CDW10 are used for specifying the type of information requested (e.g., controller or namespace structure).

  

In a Data Set Management (DSM) command, fields within the command Dwords (like CDW11) are used for specific attributes and block ranges. 

  

Therefore, to understand the exact purpose of Command Dword 4 for a specific command, you must refer to the relevant NVMe specification document for that command set (e.g., NVM Command Set, Key Value Command Set, Zoned Namespace Command Set). The official specifications are available on the NVM Express website.