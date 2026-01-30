NVMe Flush Command

The SPDK_NVME_OPC_FLUSH 0x00 command is issued by the host to ensure that any data residing in volatile memory is securely written to permanent storage. If no volatile write cache is present or enabled, the Flush command completes successfully without any effect. Once the flush operation is finished, the controller updates the associated I/O CQ with a completion entry.

  

NVMe Read Command

The SPDK_NVME_OPC_READ 0x02 command is one of the core I/O operations in NVMe. This command is issued by the host to retrieve data from a specified namespace and transfer it to the host memory.

Key parameters:

NSID – the identifier of the namespace from which the data is being read.

LBA (logical block address) – the starting address of the data to be read within the namespace.

NLB (number of LBAs) – specifies the size of the read operation in terms of the number of logical blocks to be read.

Destination buffer – The controller retrieves the read data from the namespace and sends it to the host memory, where the destination buffer is specified using PRP or SGL.

Upon completion, the controller posts a completion entry to the I/O CQ that includes a status code indicating success or failure.

  

NVMe Write Command

The SPDK_NVME_OPC_WRITE 0x01 command is one of the core I/O operations in NVMe. This command is issued by the host to write data to a specific namespace at a given LBA.

Key parameters:

NSID – identifier of the namespace where the data is being written

LBA – destination address within the namespace where data is written

NLB – specifies the size of the write operation in terms of the number of logical blocks to be written

Source buffer – the data to be written is located in host memory, and the controller reads this data from the source buffer, which is provided using PRP or SGL.

Upon completion, the controller posts a completion entry to the I/O CQ, which includes a status code indicating the success or failure of the write operation.