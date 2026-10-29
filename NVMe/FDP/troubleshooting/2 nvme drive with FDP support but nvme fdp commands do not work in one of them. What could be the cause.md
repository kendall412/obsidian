If two NVMe SSDs both advertise Flexible Data Placement (FDP) support, but `nvme fdp` commands work on only one, the most likely causes are differences in FDP enablement, Endurance Group configuration, namespace configuration, firmware implementation, or command compatibility.

The key distinction is that FDP supported does not mean FDP enabled.

## 1. Possible causes

|Cause|Explanation|
|---|---|
|1. FDP is disabled|The controller advertises FDP support through Identify Controller, but FDP has not been activated for the Endurance Group.|
|2. Incorrect Endurance Group ID|The failing command targets an EGID that does not exist or is not configured for FDP.|
|3. Different firmware versions|One drive may have incomplete FDP support or a firmware defect.|
|4. FDP configuration not activated|FDP was configured through Set Features, but the required Controller Level Reset was not performed.|
|5. Namespace not configured for FDP|Namespace-scoped commands such as RUH Status may fail because the namespace is not correctly associated with FDP placement resources.|
|6. Unsupported FDP operation|The controller may advertise FDP capability but fail to implement a required operation correctly.|
|7. Incorrect command parameters|Invalid NSID, LSI, LSP, PID, or buffer length can cause command rejection.|
|8. nvme-cli compatibility|Differences in nvme-cli/libnvme versions or command encoding can expose firmware compatibility issues.|

## 2. Run these read-only diagnostic commands

Run them on both SSDs, using the correct device path and EGID.

```
# Check firmware and FDP capability
sudo nvme id-ctrl /dev/nvme0 -H

# Retrieve available FDP configurations
sudo nvme fdp configs /dev/nvme0 -e 1

# Check whether FDP is enabled
sudo nvme fdp /dev/nvme0 -e 1

# Retrieve FDP logs
sudo nvme fdp usage /dev/nvme0 -e 1
sudo nvme fdp stats /dev/nvme0 -e 1
sudo nvme fdp events /dev/nvme0 -e 1

# Retrieve namespace RUH status
sudo nvme fdp status /dev/nvme0 -n 1
```

These commands help distinguish FDP configuration problems from namespace-specific failures. The `-e` option explicitly specifies the Endurance Group ID. An nvme-cli issue previously caused some FDP commands to use EGID 0 when it was omitted, resulting in errors.

## 3. Interpret the results

|Working commands|Failing commands|Likely problem|
|---|---|---|
|`configs`|`usage`, `stats`, `events`|FDP disabled or inactive|
|`configs`, `stats`|`status`|Namespace or RUH association|
|`configs`, `usage`, `stats`|`events`|FDP Events log implementation or LSP|
|None|All FDP commands|Incorrect EGID, firmware, or capability inconsistency|
|All generic commands|Only `nvme fdp` wrappers|nvme-cli/libnvme compatibility|

For the log commands, remember:
- LID `20h`: FDP Configurations
- LID `21h`: RUH Usage
- LID `22h`: FDP Statistics
- LID `23h`: FDP Events


These are retrieved using Admin Get Log Page, opcode `02h`.

## 4. Check the actual NVMe completion status

If a command fails, determine whether it returns:
- Invalid Field in Command: Check EGID, command DWORDs, and transfer length.
- FDP Disabled: FDP must be activated before the applicable operation can succeed.
- Invalid Log Page: Investigate log support and firmware handling.
- Invalid Namespace or Format: Verify NSID and namespace configuration.
- Timeout: Investigate controller firmware execution or PCIe/transport problems.
    
If the failing drive advertises FDP support, has FDP enabled, and rejects a valid mandatory FDP command, that is evidence of a possible firmware compliance defect.

## 5. What I would compare first

The most useful information is the output of these three commands from both drives:
```
sudo nvme fdp configs /dev/nvme0 -e 1
sudo nvme fdp /dev/nvme0 -e 1
sudo nvme fdp stats /dev/nvme0 -e 1
```

Also record `nvme version` and each drive's firmware revision.

Which exact FDP commands fail on the second SSD, and what error status do they return? For example, does `nvme fdp configs` succeed while `nvme fdp stats` fails with `Invalid Field in Command`?

That distinction is critical for determining whether the problem is FDP enablement, incorrect command parameters, or the firmware's Get Log Page implementation.