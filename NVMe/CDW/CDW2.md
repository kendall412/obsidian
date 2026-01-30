In NVMe, Command Dword 2 (CDW2) is a command-specific field within the 64-byte Submission Queue Entry structure, and its function varies depending on the specific command being executed. 

For example:

In the NVM Express Base Specification, if the data transfer for a command crosses exactly one memory page boundary, CDW2 specifies the Page Base Address of the second memory page.

In the Key Value Command Set, the Delete and Store commands use Command Dword 2 and Command Dword 3 to contain specific portions of the KV key bytes.

For the Read and Verify commands in the NVM Command Set, CDW2 and CDW3 are used for command-specific parameters related to Logical Block Addresses (LBA). 

The meaning of CDW2 is not globally fixed across all NVMe commands but is defined within the context of each command set and the specific command opcode as detailed in the relevant NVM Express specifications.