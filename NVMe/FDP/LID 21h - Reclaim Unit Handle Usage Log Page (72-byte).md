
The Reclaim Unit Handle Usage Log Page (RUH Usage Log Page) provides information about the usage classification of Reclaim Unit Handles (RUHs) associated with an FDP-enabled Endurance Group. *Its primary purpose is to help the host understand how each RUH is being used for data placement.* An important distinction is that this log page does not provide a complete inventory of physical Reclaim Units, nor does it report the amount of free space remaining in each RU.

### Why does the RUH Usage Log Page exist?

Recall the relationship between the three FDP components:
- Reclaim Group (RG): A group of Reclaim Units managed within the FDP configuration.
- Reclaim Unit Handle (RUH): A logical handle used to direct writes to Reclaim Units.
- Reclaim Unit (RU): The unit of storage managed and reclaimed by the controller.

For example:
```
FDP Configuration
|
+-- Reclaim Group 0
|   |
|   +-- RUH 0 ---> Reclaim Unit A
|   +-- RUH 1 ---> Reclaim Unit B
|   +-- RUH 2 ---> Reclaim Unit C
|
+-- Reclaim Group 1
    |
    +-- RUH 0 ---> Reclaim Unit D
    +-- RUH 1 ---> Reclaim Unit E
    +-- RUH 2 ---> Reclaim Unit F
```

The diagram illustrates how RUHs can direct writes to Reclaim Units. However, the host also needs to know how the controller classifies the usage of those RUHs. That is where LID 21h comes in.

### RUH Usage Log Page structure
The log page consists of an 8-byte header followed by one 8-byte Reclaim Unit Handle Usage Descriptor (RUHUD) for each RUH.

Log Page `21h` layout
```
Reclaim Unit Handle Usage Log Page
|
+-- Header (8 bytes)
|   |
|   +-- NRUH       (2 bytes)
|   +-- Reserved   (6 bytes)
|
+-- RUH Usage Descriptor 0 (8 bytes)
|   +-- RUHA       (1 byte)
|   +-- Reserved   (7 bytes)
|
+-- RUH Usage Descriptor 1 (8 bytes)
|   +-- RUHA       (1 byte)
|   +-- Reserved   (7 bytes)
|
+-- RUH Usage Descriptor N-1 (8 bytes)
    +-- RUHA       (1 byte)
    +-- Reserved   (7 bytes)
```


#### Header fields

|Byte offset|Size|Field|Description|
|---|---|---|---|
|00h–01h|2 bytes|NRUH|Number of Reclaim Unit Handles|
|02h–07h|6 bytes|Reserved|Set to zero|
|08h onward|8 bytes each|RUHUD|One usage descriptor per RUH|

Unlike the zero-based NCFG field discussed earlier, NRUH is an actual count. If NRUH = 8, the log contains eight RUH Usage Descriptors, numbered 0 through 7.

The total log page length is:
Log Size=8+(NRUH * 8)

For 8 RUHs:

8+(8 * 8)=72 bytes

### Reclaim Unit Handle Attributes (RUHA)
This is the most important field in the RUH Usage Log Page. Each RUHUD contains a one-byte RUHA field describing how the corresponding RUH is used.

|RUHA value|Meaning|Explanation|
|---|---|---|
|`00h`|Unused|RUH is not used by a namespace|
|`01h`|Host Specified|RUH was explicitly requested through a Placement Handle List during namespace creation|
|`02h`|Controller Specified|RUH is used for a namespace whose default handle was selected by the controller|
|`03h–FFh`|Reserved|Not valid defined usage classifications|

#### RUHA = 00h — Unused
The RUH is not currently assigned for use by a namespace. This does not necessarily mean the associated physical Reclaim Unit is empty or has no valid data.

#### RUHA = 01h — Host Specified
The host explicitly requested use of this RUH through the Placement Handle List when creating a namespace. For example, a namespace might be created with Placement Handles corresponding to RUH 0, RUH 1, and RUH 2. The log would identify those handles as host specified.

#### RUHA = 02h — Controller Specified
The namespace was created without an explicit Placement Handle List, allowing the controller to select its default RUH. The log identifies that handle as controller specified. Only one RUH Usage Descriptor may indicate controller-specified usage for the relevant Endurance Group.

##### Example: Decoding the RUH Usage Log Page
Assume the selected FDP configuration defines eight RUHs.

The controller returns the following hypothetical 72-byte log page:
```
Offset   Data
------   ------------------------
0000     08 00 00 00 00 00 00 00
0008     01 00 00 00 00 00 00 00
0010     01 00 00 00 00 00 00 00
0018     01 00 00 00 00 00 00 00
0020     02 00 00 00 00 00 00 00
0028     00 00 00 00 00 00 00 00
0030     00 00 00 00 00 00 00 00
0038     00 00 00 00 00 00 00 00
0040     00 00 00 00 00 00 00 00
```

The first two bytes, `08 00`, are little-endian and indicate NRUH = 8. Each subsequent eight-byte descriptor contains the RUHA value in its first byte.

Decoded RUH Usage Descriptors

|   |   |   |
|---|---|---|
|RUH ID|RUHA|Usage|
|0|01h|Host specified|
|1|01h|Host specified|
|2|01h|Host specified|
|3|02h|Controller specified|
|4|00h|Unused|
|5|00h|Unused|
|6|00h|Unused|
|7|00h|Unused|

From this log, the host can determine that three handles are host [[specified]], one is controller [[specified]], and four are unused. It cannot determine the amount of remaining capacity in their associated Reclaim Units.

### Relationship between LID 20h and LID 21h
These two log pages serve different purposes.

|                                    | LID `20h`                    | LID `21h`                        |
| ---------------------------------- | ---------------------------- | -------------------------------- |
| Name                               | FDP Configurations           | RUH Usage                        |
| Primary purpose                    | Describes FDP configurations | Reports RUH usage classification |
| Reports NRUH                       | Yes                          | Yes                              |
| Reports RUHA                       | No                           | Yes                              |
| Reports RU nominal size            | Yes                          | No                               |
| Describes supported configurations | Yes                          | No                               |
| Reflects namespace RUH usage       | No                           | Yes                              |

For example:
```
LID 20h - FDP Configurations
|
+-- Active FDP Configuration
    +-- NRG  = 2
    +-- NRUH = 8
             |
             v
LID 21h - RUH Usage
|
+-- RUH 0: Host Specified
+-- RUH 1: Host Specified
+-- RUH 2: Host Specified
+-- RUH 3: Controller Specified
+-- RUH 4: Unused
+-- RUH 5: Unused
+-- RUH 6: Unused
+-- RUH 7: Unused
```

One important clarification: NRUH = 8 describes eight RUH identifiers in the FDP configuration. The RUH Usage Log Page contains eight descriptors total, not eight descriptors per Reclaim Group.

The RUH identifiers can be used with Reclaim Group identifiers to form placement identifiers.

### How does the host retrieve LID 21h?

The host uses the NVMe Admin Get Log Page command, specifying:
- LID = `21h`
- The relevant Endurance Group Identifier through the Log Specific Identifier
- A buffer large enough for the returned log page
    
With `nvme-cli`, the command is:
```
nvme fdp usage /dev/nvme0 --endgrp-id=1
```

For JSON output:
```
nvme fdp usage /dev/nvme0 --endgrp-id=1 -o json
```

For the raw binary log:

```
nvme fdp usage /dev/nvme0 --endgrp-id=1 -b
```

The exact device and Endurance Group ID depend on the system. Unlike LID 20h, which supports configuration discovery before FDP is enabled, LID 21h is an FDP operational log page and is expected to return an FDP Disabled error when FDP is disabled.

### What LID 21h does not tell you
This distinction is particularly important for SSD validation.

LID 21h does not report:
- Which physical NAND blocks constitute a Reclaim Unit.
- How much free capacity remains in an RU.
- Which specific RU is currently associated with each RUH in each Reclaim Group.
- How much data has been written through a particular RUH.

For current Reclaim Unit information, the host uses the I/O Management Receive command with the Reclaim Unit Handle Status operation, rather than relying on LID 21h.

### FDP validation considerations

When validating LID 21h, useful test cases include checking that NRUH matches the active configuration's NRUH, reserved bytes are zero, every RUHA value is valid, and no more than one descriptor is controller specified.

A particularly useful functional test is to create a namespace with an explicit Placement Handle List, then retrieve LID 21h and verify that the expected handles report `RUHA = 01h`. This is also a documented FDP conformance-testing approach.

Key takeaway: LID 20h tells the host which RUHs the FDP configuration provides. LID 21h tells the host how those RUHs are being used by namespaces. It does not describe the physical Reclaim Units or their remaining capacity.