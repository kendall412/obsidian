In the NVMe command format, there is no generic "Command Dword 4 (CDW4)". Instead, the region where a theoretical CDW4 would be located is reserved in the common command structure defined by the NVMe Base Specification. 

The submission queue entry (SQE) for an NVMe command uses a common format across all commands, with specific fields having fixed purposes and others being command-specific: 

Command Dword 0 (CDW0): Contains the Opcode and Command Identifier.

Namespace Identifier (NSID): Dword 1 (4 bytes).

Dword 2 and Dword 3: These 8 bytes are reserved in the base structure. This is the location where Command Dword 4 would fall.

Metadata Pointer (MPTR): Dwords 4 and 5 (8 bytes).

Data Pointer (DPTR): Dwords 6 through 9 (16 bytes).

Command Dwords 10 through 15 (CDW10-CDW15): These 24 bytes are used for command-specific parameters. 

The specific function of Command Dword 4 (and 5) is for the Metadata Pointer (MPTR) field, which is used if metadata is transferred as a separate buffer from the main data. 

For detailed information on the specific command formats, you can refer to the official NVM Express specifications.