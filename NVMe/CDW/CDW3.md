Command DWORD 3 (CDW3) in NVMe refers to a specific 32-bit field within an NVMe command that provides additional parameters depending on the command being executed. For example, in the IDENTIFY command, CDW3 is used with the Controller or Namespace Structure (CNS) field to specify what kind of information is being requested about the controller or namespace. 

CDW fields are command-specific: Unlike a fixed instruction, the meaning of CDW3 depends entirely on the command.

Part of the command structure: NVMe commands are structured with fixed fields and variable DWORDs (like CDW1-4) that contain command-specific instructions.

Example: IDENTIFY command: The IDENTIFY command uses different CDWs to request different types of information, such as controller details or namespace data. In this case, CDW3 would be a value specifying the type of CNS (Controller or Namespace Structure) information to retrieve.