
If `nvme fdp stats`, `nvme fdp events`, or `nvme fdp usage` fails on one SSD but succeeds on another, I would first investigate Endurance Group ID (EGID), FDP enablement, and Get Log Page command handling.

All three commands use the NVMe Admin Get Log Page command (opcode 02h), but request different log pages.

## 1. Understand what each command does

|Command|Log Page ID|Description|
|---|---|---|
|`nvme fdp usage`|21h|Reclaim Unit Handle Usage|
|`nvme fdp stats`|22h|FDP Statistics|
|`nvme fdp events`|23h|FDP Events|

These logs are Endurance Group scoped. The EGID is supplied through the Log Specific Identifier (LSI) field. The Events log additionally uses LSP to select the host or controller event log.

## 2. First, run these commands on both drives

Use the actual EGID rather than assuming it is 1.

```
# Check FDP capability
sudo nvme id-ctrl /dev/nvme0 -H

# Check FDP configuration
sudo nvme fdp configs /dev/nvme0 -e 1

# Check FDP enablement
sudo nvme fdp /dev/nvme0 -e 1

# Retrieve RUH usage
sudo nvme fdp usage /dev/nvme0 -e 1

# Retrieve FDP statistics
sudo nvme fdp stats /dev/nvme0 -e 1

# Retrieve controller events
sudo nvme fdp events /dev/nvme0 -e 1

# Retrieve host events
sudo nvme fdp events /dev/nvme0 -e 1 -E
```

The `-e` argument specifies the Endurance Group ID, and `-E` selects host events.

A documented nvme-cli issue involved FDP Statistics and Events commands defaulting to EGID 0 when no EGID was provided, causing failures. Explicitly supplying `-e` eliminates that particular ambiguity.


## 3. Identify the failure pattern

|Result|Most likely area to investigate|
|---|---|
|Configs succeeds; all three other logs fail|FDP disabled or incorrect EGID|
|Configs and all other logs fail|EGID, Get Log Page encoding, capability advertisement|
|Usage fails; Stats and Events succeed|LID 21h implementation or variable-length response|
|Stats fails; Usage and Events succeed|LID 22h implementation or transfer length|
|Events fails; Usage and Stats succeed|LID 23h, LSP, or event-log formatting|
|Controller Events works; Host Events fails|LSP-specific firmware handling|
|Commands succeed but show empty data|May be valid; inspect log header and current state|

Important: An empty FDP Events log can be normal. No recorded events is different from the Get Log Page command failing.

## 4. Check whether FDP is enabled

For the affected Endurance Group:

```
sudo nvme fdp /dev/nvme0 -e 1
```

Compare the current FDP enable state between the working and failing drives.

A controller may allow the FDP Configurations log (20h) to be retrieved while FDP is disabled, but reject the active FDP Usage, Statistics, and Events logs with FDP Disabled. This behavior is also illustrated by an open-source FDP controller implementation.

Do not change FDP enablement yet. First establish whether the failure is expected because FDP is disabled.

## 5. Distinguish nvme-cli problems from firmware problems

If FDP is enabled and the EGID is correct, the next step is to test the underlying Get Log Page command.

Check your installed version:

```
nvme version
nvme fdp stats --help
nvme get-log --help
```

For example, retrieve the fixed-size FDP Statistics log using the generic Get Log Page interface:

```
sudo nvme get-log /dev/nvme0 \
    --log-id=0x22 \
    --log-len=64 \
    --lsi=1 \
    --raw-binary
```

This requests LID 22h with a 64-byte transfer and LSI 1. The generic Get Log Page interface supports explicit log ID, transfer length, and related fields.

If the generic command succeeds while `nvme fdp stats` fails, investigate nvme-cli or libnvme argument encoding and response parsing.

If both fail with the same NVMe completion status, focus on the controller's command validation, FDP state, and firmware log-page handler.

For Usage and Events, first obtain the required response size from their headers or specification-defined structures rather than assuming a fixed length.

## 6. Decode the NVMe completion status

The exact status determines the next debugging step.

|NVMe status|Investigation|
|---|---|
|Invalid Field in Command|EGID/LSI, LSP, NUMD, or other command fields|
|Invalid Log Page|Log not implemented, not available, or invalid request|
|FDP Disabled|FDP not active for the specified Endurance Group|
|Invalid Opcode|Admin command dispatch or incorrect opcode|
|Command timeout|Firmware hang, transport, or command processing deadlock|
|Successful completion with invalid data|Firmware log construction or host parser|

For firmware-level debugging, capture the 64-byte Admin Submission Queue Entry and completion status.

The most relevant Get Log Page fields are:
- CDW10: LID, LSP, RAE, NUMDL
- CDW11: NUMDU
- CDW12–13: Log Page Offset
- CDW14: Log Specific Identifier (LSI), containing the EGID for these FDP logs


Compare the raw SQEs between the working and failing drives. Verify that the commands are equivalent and that the EGID and transfer lengths are valid for each controller.

## 7. Firmware debugging checkpoints

If the same valid command succeeds on one drive and fails on another, inspect these firmware stages:

1 Admin Get Log Page dispatch (opcode 02h)

2 LID validation (21h / 22h / 23h)

3 Endurance Group ID validation

4 FDP enabled-state check

5 LSP and transfer-length validation

6 Log data construction and DMA transfer

7 Completion status generation

For example, if LID 20h works but LIDs 21h, 22h, and 23h all return Invalid Field in Command, examine whether the firmware rejects those logs when FDP is disabled or incorrectly validates the EGID.

If only LID 23h fails, inspect host/controller event selection through LSP and the event-log buffer construction.

## 8. Information needed to isolate your failure

Run these three commands on the failing drive and capture the complete output:
```
sudo nvme fdp configs /dev/nvme0 -e 1
sudo nvme fdp /dev/nvme0 -e 1
sudo nvme fdp stats /dev/nvme0 -e 1
```

Then provide the exact error message, particularly the SCT and SC if available.

The key diagnostic question is whether all three FDP logs fail, or only one specific log. That distinction will tell us whether to focus on shared FDP-state validation or a particular firmware Get Log Page handler.
