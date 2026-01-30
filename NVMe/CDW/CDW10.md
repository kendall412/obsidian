Command DWORD 10 (CDW10) in NVMe commands typically contains a queue identifier and queue size for commands that create or manage I/O queues. The specific meaning of CDW10 depends on the command being issued; for example, in the Create I/O Completion Queue command, CDW10 specifies the queue ID and size. For other commands like the Identify command, CDW10 holds a different value that indicates the type of information to be returned (like Controller or Namespace data). 

Command DWORD 10 (CDW10) details by command 

Create I/O Completion Queue: This is the most common use case for the terms "queue identifier" and "queue size."

Low word: Queue Identifier (Queue ID).

High word: Queue Size, given as one less than the actual number of entries.

Identify Command: CDW10 contains a value that specifies which type of information will be returned in the command's data buffer.

Example: A value in CDW10 might tell the controller to return the Controller Structure or a Namespace Structure.

Format NVMe Command: CDW10 is reserved and should be initialized to zero for this command.

Other Commands: The meaning of CDW10 varies by command. You must refer to the specific command's definition in the NVMe specification to understand what information it carries.