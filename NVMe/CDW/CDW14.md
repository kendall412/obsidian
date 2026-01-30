![[Pasted image 20260129210349.png]]

Command Dword (CDW) 14 in NVM Express is part of the general command structure and is used to specify the higher-order bytes (bits 15:8) of a Key-Value (KV) key, specifically for commands that interact with KV namespaces. It works in conjunction with CDW2, CDW3, and CDW15 to define the full KV key. 

  

Purpose: Specifies the upper part of a Key-Value key.

  

Context: It is used in the Key-Value Command Set specification for commands like Get, Delete, and List.

  

Function: CDW14 is one of four command dwords used to construct the 16-byte KV key, with CDW2 and CDW3 handling the lower bytes and CDW15 handling the remaining higher bytes.

  

Example: A command using the KV command set will use CDW2, CDW3, CDW14, and CDW15 to provide the full KV key for operations such as creating, reading, or deleting a key-value pair.