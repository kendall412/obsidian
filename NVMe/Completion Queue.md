Completion Queue

The completion queue (CQ) in NVMe stores entries that the controller writes after processing commands. Each entry in the CQ corresponds to a command submitted through a submission queue (SQ), and the host checks the CQ to track the status of these commands.

CQs are implemented as circular buffers in host memory. The host can either poll the CQ or use interrupts to be notified when new entries are available.

Completion Queue Element

Each completion queue element (CQE) in an NVMe CQ is an individual entry that contains status information about a completed command, including:

CID – Identifier of the completed command

SQID – ID of the SQ from which the command was issued

SQHD – Marks the point in the SQ up to which commands have been completed

SF – Indicates the status of the completed command

P – Flags whether the completion entry is new