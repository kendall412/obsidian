
It is a field in the Identify Controller data structure that tells the host which optional NVMe NVM commands or related capabilities the controller supports.

If a bit in ONCS is set to 1, that feature or command is supported. If it is 0, the host should not issue that optional command or use that optional capability.

Location
In the NVMe Identify Controller data structure:

|Field|Meaning|
|---|---|
|`ONCS`|Optional NVM Command Support|
|Size|16 bit|
|Offset|Commonly byte `520` in Identity Controller data|
|Endianness|Little Endian|

### ONCS bit definitions
|Bit|Name|Meaning|
|---|---|---|
|`0`|Compare|Controlelr supports the Compare command|
|`1`|Write Uncorrected|Controller supports Write Uncorrectable|
|`2`|Dataset Mgmt|Controlelr supports Dataset Mgmt, including deallocate/TRIM|
|`3`|Write Zeroes|Contoller supports Write Zeroes|
|`4`|Save/Select Features|Controller supports Save field in Set Features and Select field in Get Features|
|`5`|Reservations|Controller supports NVMe Reservations|
|`6`|Timestamp|Contoller supports Timestamp feature, FID `0Eh`|
|`7`|Verify|Controller supports the Verify command|
|`8`|Copy|Controlelr supports Copy command|
|`15:9`|Reserved|Reserved, normally zero|

Exact supported bits can depend on the NVMe specification version implemented by the device.

### Bit details<br>
#### Bit 0 — Compare command
If set:
```
ONCS[0] = 1
```
The controller supports the **Compare** command.

The Compare command compares logical block data on the drive with data supplied by the host. It is commonly used to verify that data on media matches an expected buffer. If not supported, issuing a Compare command may return an invalid opcode or invalid command error.

#### Bit 1 — Write Uncorrectable<br>
If set:
```
ONCS[1] = 1
```
The controller supports the **Write Uncorrectable** command.

This command marks one or more logical blocks as containing unrecoverable data. Future reads to those LBAs may return an unrecovered read error.

Typical uses include:

- Testing error handling
- Marking defective or invalid logical blocks
- Storage validation workflows

#### Bit 2 — Dataset Management<br>
If set:
```
ONCS[2] = 1
```
The controller supports the **Dataset Management** command.

The <u>most common Dataset Management operation is deallocate</u>, often known as:
```
TRIM or discard
```
This allows the host to tell the SSD that certain logical blocks are no longer in use.

Benefits include:

- Better garbage collection
- Improved write performance
- Reduced write amplification
- More efficient flash management

Example Linux command that may use this capability:
```
fstrim
```

#### Bit 3 — Write Zeroes<br>
If set:
```
ONCS[3] = 1
```
The controller supports the **Write Zeroes** command.

This command causes a range of logical blocks to read back as zeroes.

It may be used for:

- Fast block clearing
- Thin provisioning
- Filesystem initialization
- Virtual machine image preparation

Depending on controller behavior and command options, Write Zeroes may or may not physically write zeroes to media.

#### Bit 4 — Save and Select fields<br>
If set:
```
ONCS[4] = 1
```
The controller supports:

- The `Save`, or `SV`, field in `Set Features`
- The `Select`, or `SEL`, field in `Get Features`

This allows the host to work with different feature values such as:

| Get Feature `SEL` Value | Meaning                |
| ----------------------- | ---------------------- |
| `0`                     | Current value          |
| `1`                     | Deafult value          |
| `2`                     | Saved value            |
| `3`                     | Supported capabilities |
If this bit is not set, the host should not assume saved feature values or selectable feature queries are supported.

#### Bit 5 — Reservations
#reservations
If set:
```
ONCS[5] = 1
```
The controller supports NVMe **Reservations**.

NVMe Reservations are used for shared-storage coordination between multiple hosts. They are similar in purpose to SCSI persistent reservations.

Commands related to this include:

- Reservation Register
- Reservation Acquire
- Reservation Release
- Reservation Report

Typical use cases:

- Clustering
- Multipath storage
- Failover systems
- Shared NVMe namespaces

#### Bit 6 — Timestamp
#timestamp
If set:
```
ONCS[6] = 1
```
The controller supports the **Timestamp** feature.

This corresponds to NVMe Feature Identifier:
```
FID = 0Eh
```

The host can use:
```
Get Features, FID 0Eh
```

to read the controller timestamp, and:
```
Set Features, FID 0Eh
```

to set it.

The timestamp is typically used for:

- Event logs
- Persistent event logs
- Telemetry
- Debugging
- Correlating controller events with host time

#### Bit 7 — Verify<br>
If set:
```
ONCS[7] = 1
```
The controller supports the **Verify** command.

The Verify command checks that logical blocks are readable and valid without transferring the data to the host.

It can be used for:

- Media verification
- Data integrity checking
- Background validation
- Maintenance operations

#### Bit 8 — Copy<br>
If set:
```
ONCS[8] = 1
```
The controller supports the **Copy** command.

The Copy command allows the device to copy data internally from one set of logical block ranges to another, reducing host data movement.

This can improve performance for operations such as:

- File cloning
- Snapshot-like operations
- Data migration inside the same namespace/device
- Offloaded copy

### Example ONCS decoding
Suppose Identify Controller reports:
```
oncs    : 0x5f
```

Binary:
```
0x5f = 0101 1111b
```

Bits set:
```
0, 1, 2, 3, 4, 5, 6
```

So the controller supports:

- Compare
- Write Uncorrectable
- Dataset Management
- Write Zeroes
- Save/Select Features
- Timestamp

But does not support:

- Reservations, bit 5
- Verify, bit 7
- Copy, bit 8

Some versions of `nvme-cli` also decode ONCS in human-readable form using:
```
nvme id-ctrl /dev/nvme0 -H
```
### Important notes<br>
- `ONCS` is reported by the controller, not by an individual namespace.
- A set bit means the optional command or capability is supported by the controller.
- A cleared bit means the host should not use that optional command.
- Some capabilities may also have namespace-specific behavior or additional limitations.
- Reserved bits should be ignored by software.
- The meaning of newer bits can depend on the NVMe specification version.