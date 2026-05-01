

NVMe Read Command

The SPDK_NVME_OPC_READ 0x02 command is one of the core I/O operations in NVMe. This command is issued by the host to retrieve data from a specified namespace and transfer it to the host memory.

Key parameters:

NSID – the identifier of the namespace from which the data is being read.

LBA (logical block address) – the starting address of the data to be read within the namespace.

NLB (number of LBAs) – specifies the size of the read operation in terms of the number of logical blocks to be read.

Destination buffer – The controller retrieves the read data from the namespace and sends it to the host memory, where the destination buffer is specified using PRP or SGL.

Upon completion, the controller posts a completion entry to the I/O CQ that includes a status code indicating success or failure.

  
