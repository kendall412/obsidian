
In NVMe Flexible Data Placement (FDP), you can create a namespace with specific Placement Handles (PHNDLs) by providing a Placement Handle List as part of the Namespace Management Create command.

The important distinction is:

- PHNDL (Placement Handle): Identifies a Reclaim Unit Handle (RUH) within the FDP configuration.
- PID (Placement Identifier): Combines a Reclaim Group Identifier (RGID) and a Placement Handle.
- Placement Handle List: Specifies which Placement Handles are associated with the namespace.
    
You do not directly configure a complete PID during namespace creation. Instead, you specify the Placement Handles that the namespace may use. The RGID is incorporated when forming a PID for subsequent FDP operations.

## 1. Example configuration

Assume the controller advertises:

|Parameter|Value|
|---|---|
|FDP enabled|Yes|
|Reclaim Groups (NRG)|2|
|Reclaim Unit Handles (NRUH)|8|
|RGIF|1|
|Desired namespace Placement Handles|2, 4, 6|

The objective is to create a namespace with access to Placement Handles 2, 4, and 6.

After creation, the valid PIDs associated with those handles would be:

|RGID|PHNDL|PID|
|---|---|---|
|0|2|0x0002|
|0|4|0x0004|
|0|6|0x0006|
|1|2|0x8002|
|1|4|0x8004|
|1|6|0x8006|

These values assume the specified RGIF and that the namespace's Placement Handle List contains the indicated handles.

## 2. Important: Placement Handles are assigned sequentially

There is an important correction to the example above.

According to NVMe Base Specification 2.2, Placement Handles are assigned sequentially starting at 0, and each entry in the Placement Handle List specifies which RUH that handle references.

![](https://www.google.com/s2/favicons?domain=https://nvmexpress.org&sz=32)

NVM Express

For example, if you create a namespace with this list:

```
Placement Handle List = [2, 4, 6]
```

The resulting mapping is:

|Placement Handle|RUH Identifier|
|---|---|
|PHNDL 0|RUH 2|
|PHNDL 1|RUH 4|
|PHNDL 2|RUH 6|

Therefore, the namespace has PHNDLs 0, 1, and 2—not PHNDLs 2, 4, and 6.

With RGIF = 1, the corresponding PIDs are:

|RGID|PHNDL|PID|Selected RUH|
|---|---|---|---|
|0|0|0x0000|2|
|0|1|0x0001|4|
|0|2|0x0002|6|
|1|0|0x8000|2|
|1|1|0x8001|4|
|1|2|0x8002|6|

## 3. Create the namespace using nvme-cli

Assuming FDP is already enabled for Endurance Group 1 and RUH identifiers 2, 4, and 6 are valid:

```
sudo nvme create-ns /dev/nvme0 \
    --nsze=1048576 \
    --ncap=1048576 \
    --flbas=0 \
    --endg-id=1 \
    --csi=0 \
    --nphndls=3 \
    --phndls=2,4,6
```

The important parameters are:

|Parameter|Meaning|
|---|---|
|`--nsze`|Namespace size in logical blocks|
|`--ncap`|Namespace capacity in logical blocks|
|`--flbas`|Selected LBA format|
|`--endg-id=1`|Target Endurance Group|
|`--csi=0`|NVM Command Set|
|`--nphndls=3`|Three Placement Handles|
|`--phndls=2,4,6`|RUH identifiers assigned to PHNDLs 0, 1, and 2|

These options are supported by nvme-cli.

![](https://www.google.com/s2/favicons?domain=https://nvme-cli.readthedocs.io&sz=32)

nvme-cli.readthedocs.io

Warning: Namespace management changes storage configuration and may destroy data. Use an unmounted test device, confirm the correct controller and Endurance Group, and ensure the operation is authorized.

## 4. Attach and verify the namespace

After creation, the controller returns the newly assigned NSID. Assuming NSID 1 and controller ID 1:

```
sudo nvme attach-ns /dev/nvme0 \
    --namespace-id=1 \
    --controllers=1
```

Then retrieve the RUH status:

```
sudo nvme fdp status /dev/nvme0 \
    --namespace-id=1
```

Verify the returned PID-to-RUH mappings.

## 5. Creating a namespace with a particular PID

Suppose you want to write using PID `0x0002`, and you want that PID to select RUH 6.

Create the namespace with three Placement Handles:

```
sudo nvme create-ns /dev/nvme0 \
    --nsze=1048576 \
    --ncap=1048576 \
    --flbas=0 \
    --endg-id=1 \
    --csi=0 \
    --nphndls=3 \
    --phndls=2,4,6
```

This establishes:

```
PID 0x0000 → PHNDL 0 → RUH 2
PID 0x0001 → PHNDL 1 → RUH 4
PID 0x0002 → PHNDL 2 → RUH 6
```

The example assumes RGID 0.

Key point: You cannot arbitrarily assign PHNDL numbers during namespace creation. You control the number of Placement Handles and the RUH identifier associated with each sequentially numbered Placement Handle.

If you want a specific RUH to be selected by PID `0x0002`, put that RUH identifier in the third entry of the Placement Handle List.