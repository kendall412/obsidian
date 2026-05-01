

The SPDK_NVME_OPC_FLUSH 0x00 command is issued by the host to ensure that any data residing in volatile memory is securely written to permanent storage. If no volatile write cache is present or enabled, the Flush command completes successfully without any effect. Once the flush operation is finished, the controller updates the associated I/O CQ with a completion entry.

  

