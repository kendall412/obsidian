NVMe Write Command

The SPDK_NVME_OPC_WRITE 0x01 command is one of the core I/O operations in NVMe. This command is issued by the host to write data to a specific namespace at a given LBA.

Key parameters:

NSID – identifier of the namespace where the data is being written

LBA – destination address within the namespace where data is written

NLB – specifies the size of the write operation in terms of the number of logical blocks to be written

Source buffer – the data to be written is located in host memory, and the controller reads this data from the source buffer, which is provided using PRP or SGL.

Upon completion, the controller posts a completion entry to the I/O CQ, which includes a status code indicating the success or failure of the write operation.