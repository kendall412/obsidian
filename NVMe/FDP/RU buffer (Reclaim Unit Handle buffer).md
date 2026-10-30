
> In NVMe Flexible Data Placement (FDP), a Reclaim Unit Handle (RUH) buffer generally refers to a data buffer containing information about Reclaim Unit Handles that the host sends to or receives from the NVMe controller. However, "RUH buffer" is not a single, universally defined NVMe structure. Its meaning depends on the command being used.

In NVMe Base Specification 2.2, there are three important contexts.

### 1. Types of buffers associated with RUHs

|Buffer|NVMe command|Purpose|
|---|---|---|
|RUH Status data buffer|I/O Management Receive|Returns RUH status information|
|RUH Update data buffer|I/O Management Send|Specifies PIDs whose associated RUHs are to be updated|
|Placement Handle List|Namespace Management (Create)|Specifies RUH identifiers associated with namespace Placement Handles|

These buffers serve different purposes.

### 2. RUH Status data buffer

The host uses I/O Management Receive, with the Reclaim Unit Handle Status management operation, to retrieve the current state of RUHs. 

The controller fills a host-provided data buffer with RUH status information. 

![[host provided data buffer.png]]

The returned data contains a header and RUH status descriptors.

Important fields include:

|Field|Description|
|---|---|
|NRUHSD|Number of RUH Status Descriptors|
|PID|Placement Identifier associated with the RUH|
|RUHID|Reclaim Unit Handle Identifier|
|EARUTR|Estimated Active Reclaim Unit Time Remaining|
|RUAMW|Reclaim Unit Available Media Writes|

The status buffer lets the host determine which placement identifiers are available and how much media-write capacity remains in their associated active Reclaim Units.

#### Example

Suppose the controller returns:

|PID|RUHID|RUAMW|
|---|---|---|
|0x0000|0|1,048,576|
|0x0001|1|524,288|
|0x0002|2|262,144|

These values are illustrative. The host can use the information to monitor placement resource consumption.

### 3. RUH Update data buffer

The host uses I/O Management Send, with the Reclaim Unit Handle Update management operation, to request an update of RUHs associated with specified PIDs.

The host constructs a data buffer containing the PID list.

For example:

```
RUH Update Data Buffer

PID 0 = 0x0001
PID 1 = 0x0002
PID 2 = 0x0003
```

Conceptually:

![[host to ctrl ruh update.png]]

The controller interprets the PIDs and updates the applicable RU associations according to the RUH Update operation's rules.

The host does not directly supply physical NAND block addresses or choose a particular physical Reclaim Unit.

## 4. Placement Handle List buffer

A third related structure is the Placement Handle List used during Namespace Management Create.

Suppose the host wants three namespace Placement Handles mapped to RUH identifiers 2, 4, and 6.

The list is conceptually:

```
Placement Handle List

Entry 0 → RUHID 2
Entry 1 → RUHID 4
Entry 2 → RUHID 6
```

The namespace's Placement Handles are numbered sequentially from zero:

|PHNDL|RUHID|
|---|---|
|0|2|
|1|4|
|2|6|

This is different from the RUH Update buffer, which identifies placement selections to update.

## 5. How these buffers relate to SSD firmware

|Operation|Data direction|What firmware does|
|---|---|---|
|RUH Status|Controller → Host|Builds descriptors from current RUH state|
|RUH Update|Host → Controller|Parses PID list and processes RUH updates|
|Namespace Create|Host → Controller|Establishes namespace Placement Handle mapping|

For validation, it is important to test both the command structure and the data buffer itself, including descriptor counts, valid and invalid identifiers, transfer lengths, reserved fields, and data integrity.

Key takeaway: An RUH buffer is generally host-memory data used to communicate RUH information through NVMe commands. It should not be confused with the Reclaim Unit itself, which represents physical storage managed by the controller.

If you are referring specifically to the RUH Update buffer in I/O Management Send, I can explain its exact byte offsets, DWORD layout, and how to construct it in Python according to NVMe Base Specification 2.2.