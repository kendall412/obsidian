
The controller can report its FDP capabilities and configurations even while FDP is disabled.
### Where do FDP Configuration Descriptors come from?
The SSD manufacturer defines the supported FDP configurations through the controller's firmware and underlying hardware capabilities. FDP configurations are already defined by the SSD controller's firmware before FDP is enabled. More precisely, the firmware contains the logic and configuration information needed to report the supported FDP configurations.

These configurations may depend on:
- NAND architecture: Number of dies, channels, and physical storage resources.
- Controller architecture: How the controller manages reclaim units and reclaim groups.
- Firmware design: Supported placement identifiers, reclaim group arrangements, and FDP resource management.
- Endurance Group configuration: The FDP configurations associated with the specified Endurance Group.

The controller firmware exposes these configurations through the FDP Configurations Log Page (LID `20h`).

The descriptors may be stored in firmware tables, constructed from internal configuration data, or assembled dynamically when the log page is requested. The NVMe specification defines the information returned, not the manufacturer's internal implementation.
#### Example:
An SSD controller's firmware might support:

```
SSD Controller Firmware
|
+-- FDP Configuration 0
|   +-- 1 Reclaim Group
|   +-- 4 Reclaim Unit Handles
|
+-- FDP Configuration 1
|   +-- 2 Reclaim Groups
|   +-- 8 Reclaim Unit Handles
|
+-- FDP Configuration 2
    +-- 4 Reclaim Groups
    +-- 16 Reclaim Unit Handles
```

When the host issues Get Log Page (LID `20h`), the controller reports the supported configurations regardless of whether FDP is enabled.

When FDP is subsequently enabled, the controller uses the selected supported configuration. The configurations are firmware-defined, but their implementation must be compatible with the SSD's hardware resources, NAND organization, and controller architecture.

In other words, FDP enablement activates existing controller capabilities rather than creating new FDP capabilities.


#### Example:
Suppose an SSD manufacturer implements firmware supporting these illustrative configurations:

|Parameter|Configuration 0|Configuration 1|
|---|---|---|
|Reclaim Groups|1|2|
|Reclaim Unit Handles|4|8|
|Supported Placement Identifiers|4|16|

These values are hypothetical, not prescribed by NVMe.

When the SSD powers on:
1. The controller initializes its internal configuration information
2. FDP remains disabled
3. The host requests LID `20h`
4. The controller returns the supported FDP Configuration Descriptors
5. The host examines the available configurations
6. The host may enable FDP using a supported configuration

Enabling FDP does not mean the controller must generate a new configuration descriptor. Instead, the controller begins operating according to the selected configuration.

### What changes when FDP becomes enabled?

|Item|FDP disabled|FDP enabled|
|---|---|---|
|FDP Configuration Descriptors|Available|Available|
|FDP placement operation|Inactive|Active|
|Reclaim Unit Handles|Described by configuration|Used for FDP operation|
|Placement Identifiers|Supported values can be determined|Used to direct write placement|
|FDP accounting and reclaim behavior|Not active for FDP placement|Active as specified|
*The FDPE field is associated with an individual FDP Configuration Descriptor, not a single global enabled flag for every descriptor.*

Therefore:
- Before FDP is enabled, no configuration descriptor indicates that it is enabled.
- After FDP is enabled, the descriptor corresponding to the active configuration indicates that it is enabled.
- Other supported configuration descriptors remain available.

Bottom line: Enabling FDP generally changes which configuration is marked active in LID 20h. It does not require changing the underlying supported configuration parameters.

Other FDP log pages, such as the Reclaim Unit Handle Usage and FDP Statistics log pages, provide information about operational state rather than merely describing supported configurations.
