# Set Feature Command
## NVMe Base Specification 5.1.25 pg 367

The Set Features command in NVMe is **executed synchronously by the host driver whenever it needs to change a configurable attribute of the controller or a specific namespace**. Unlike some background maintenance tasks, it does not happen automatically on a timer; **it is always triggered by a specific event or condition in the host software stack**.

**Here are the specific scenarios and timing when the Set Features command gets executed**:

1. During Driver Initialization (Boot Time)

    This is the most common execution time. When the OS loads the NVMe driver (e.g., `nvme.ko` in Linux or `nvme.sys` in Windows), it performs a sequence to configure the drive for optimal operation.

    - Timing: Immediately after the driver identifies the controller (`Identify Controller`) and before it starts submitting regular I/O (Read/Write) commands.
    - Common Features Set:
        - Interrupt Coalescing: To balance CPU usage and latency.
        - Error Recovery: Setting timeouts (e.g., "If an error occurs, retry for 5 seconds before failing").
        - Write Cache: Enabling or disabling the volatile write cache (critical for data integrity).
        - [[Fault Tolerance Policy (FTP)]]: As discussed previously, setting the desired error handling behavior.
        - Queue Depth: Configuring the number of submission/completion queues.

2. During Runtime Configuration (Dynamic Changes)

    **The host may execute Set Features while the system is running if an administrator or application requests a change in behavior**.

    - Timing: Triggered by a user command or a system policy change.
    - Examples:
        - Admin Tool Usage: When a user runs a command like nvme set-feature in Linux or uses a vendor-specific GUI tool to change power settings or temperature thresholds.
        - Power Management: The OS power manager might switch the drive to a lower power state (e.g., APST - - Autonomous Power State Transition) when the system is idle, then switch it back to high performance when load increases.
        - Thermal Throttling: If the OS detects high temperatures, it might lower the thermal threshold feature to force the drive to slow down sooner.

3. During Error Recovery or Reset Sequences

    If the drive encounters a severe error or the driver detects a timeout, the driver may attempt to recover the device without a full system reboot.

    - Timing: Immediately after a Controller Reset or Namespace Reset.
    - Process:
        - Driver detects a hang or fatal error.
        - Driver issues a Controller Reset.
        - Once the controller comes back online, the driver re-executes Set Features to restore the configuration (since many features are volatile and reset to defaults after a controller reset).
        - Normal I/O resumes.

4. How the Execution Happens (The Mechanism)

    It is important to understand that **Set Features is a blocking operation from the driver's perspective**.

    1. Submission: The driver places the Set Features command into the Admin Submission Queue (ASQ). This is a special queue reserved for management commands, separate from the I/O queues used for reading/writing data.
    2. Doorbell Ring: The driver writes to the "Doorbell" register in the PCIe memory space to notify the NVMe controller that a new command is waiting.
    3. Execution: The NVMe controller fetches the command, processes it internally (e.g., updates a register or changes an internal state machine), and prepares a response.
    4. Completion: The controller places a completion entry in the Admin Completion Queue (ACQ) and triggers an interrupt (or the driver polls for it).
    5. Verification: The driver reads the completion entry.
        - If Success: The driver proceeds, knowing the feature is now active.
        - If Failed: The driver logs an error and may fall back to a default setting or abort the initialization.

## Summary of Timing

|Scenario|Trigger Event|Critically|
|---|---|---|
|Boot/Load|OS Kernel loads NVMe driver|High: Required for basic operation|
|Resume from Sleep|System wakes from S3/S4 sleep states|High: features ofter reset during sleep|
|Admin Action|Users runs `nvme-cli` or management software|Medium: Optional tuning|
|Recovery|Drive reset after a crash or timeout|High: Must restore config to function|
|Power State Change|OS transitions btwn Performance/Idle modes|Medium: Optimizes power/heat|

## Key Distinction: Volatile vs. Non-Volatile

- Volatile (Most Common): **Most `Set Features` commands (like Interrupt Coalescing, FTP, Write Cache) only last until the next Controller Reset or power cycle**. The driver must re-execute them every time the system boots or the drive resets.
- Non-Volatile (Rare): A few features (like certain Namespace configurations) can be saved to non-volatile storage on the drive if the "Save" bit is set in the command. These persist across reboots, but the driver usually still re-sends them to ensure the state is known and correct.

**In the context of FTP (Fault Tolerance Policy), the Set Features command is almost always executed during driver initialization (boot) to ensure the drive behaves correctly from the moment the OS takes control**. If the drive resets due to an error, the driver will re-send this command during the recovery phase to re-establish the safety policy.

## NVMe Base Spec v2.2 5.1.25 Set Feature Command
The Set Features command uses the Data Pointer, Command Dword 10, and Command Dword 14. The use of Command Dword 11, Command Dword 12, Command Dword 13, and Command Dword 15 fields is Feature specific. If Command Dword 11, Command Dword 12, Command Dword 13, or Command Dword 15 fields are not used, then the Command Dwords are reserved.

|Bits|Descriptiom|
|:-:|---|
|127:0|Data Pointer (DPTR): This field specifies the start of the data buffer. Refer to Figure 92 for the definition of this field. If using PRPs, this field shall not be a pointer to a PRP List as the data buffer may not cross more than one page boundary. If no data structure is used as part of the specified feature, then this field is not used.|
|31|Save (SV): This bit specifies that the controller shall save the attribute so that the attribute persists through all power states and resets. The controller indicates in the Save and Select Feature Support (SSFS) bit in the Optional NVM Command Support field of the Identify Controller data structure in Figure 313 whether this bit is supported. If the Feature Identifier specified in the Set Features command is not saveable by the controller and the controller receives a Set Features command with this bit set to ‘1’, then the command shall be aborted with a status code of Feature Identifier Not Saveable.|
|30:08|Reserved|
|07:00|Feature Identifier (FID): This field indicates the identifier of the Feature that attributes are being specified for.|


```
sudo nvme set-feature /dev/nvme0 -f 0x0e -V 0x10c1d15c00 --save
```
### Parameters Explained

* `/dev/nvme0`: The target NVMe character device or controller.
* `-f 0x0e`: Specifies the Feature ID (FID) in hex (`0xE`).
* `-V <value>`: Sets the data value for Command Dword 11 (replace `<value>` with the specific numeric setting required for your device).
* `-s` (or `--save`): Ensures the attribute persists through power states and system resets, provided your drive supports the Save and Select Feature.

## Features available for Set-Feature cmd

Below are the NVMe **Set Features** Feature Identifier values defined/assigned in the **NVMe Base Specification 2.2** feature table. Actual support is controller/namespace dependent.

| FID | Feature |
|---:|---|
| `0x01` | Arbitration |
| `0x02` | Power Management |
| `0x03` | LBA Range Type |
| `0x04` | Temperature Threshold |
| `0x05` | Error Recovery |
| `0x06` | Volatile Write Cache |
| `0x07` | Number of Queues |
| `0x08` | Interrupt Coalescing |
| `0x09` | Interrupt Vector Configuration |
| `0x0A` | Write Atomicity Normal |
| `0x0B` | Asynchronous Event Configuration |
| `0x0C` | Autonomous Power State Transition |
| `0x0D` | Host Memory Buffer |
| `0x0E` | Timestamp |
| `0x0F` | Keep Alive Timer |
| `0x10` | Host Controlled Thermal Management |
| `0x11` | Non-Operational Power State Config |
| `0x12` | Read Recovery Level Config |
| `0x13` | Predictable Latency Mode Config |
| `0x14` | Predictable Latency Mode Window |
| `0x15` | LBA Status Information Attributes / Report Interval |
| `0x16` | Host Behavior Support |
| `0x17` | Sanitize Config |
| `0x18` | Endurance Group Event Configuration |
| `0x19` | I/O Command Set Profile |
| `0x1A` | Spinup Control |
| `0x1B` | Power Loss Signaling Config |
| `0x1C` | Performance Characteristics |
| `0x1D` | Flexible Data Placement |
| `0x1E` | Flexible Data Placement Events |
| `0x1F` | Namespace Refresh |
| `0x7D` | Enhanced Controller Metadata |
| `0x7E` | Controller Metadata |
| `0x7F` | Namespace Metadata |
| `0x80` | Software Progress Marker |
| `0x81` | Host Identifier |
| `0x82` | Reservation Notification Mask |
| `0x83` | Reservation Persistence |
| `0x84` | Namespace Write Protection |
| `0xC0–0xFF` | Vendor Specific |

Reserved/unassigned ranges include values such as `0x00`, `0x20–0x7C`, and `0x85–0xBF`.
 ---
 ## 64-Byte Submission Queue Entry (SQE) Layout

 For the NVMe **Set Features** command with **FID 0xE (Interrupt Coalescing)**, here is the 64-byte Submission Queue Entry (SQE) data structure:

| Offset (Bytes) | Field | Size | Value for FID 0xE | Description |
| :--- | :--- | :--- | :--- | :--- |
| **00-01** | **Opcode** | 2 B | `0x09` | Set Features command opcode. |
| **02** | **Flags** | 1 B | `0x00` | Usually 0 for Set Features. |
| **03** | **PSDT** | 1 B | `0x00` | PRP/SG List Descriptor Type. |
| **04-05** | **CID** | 2 B | `Unique` | Command Identifier. |
| **08-11** | **NSID** | 4 B | `0x00000000` | Namespace ID. **Must be 0** for global features like Interrupt Coalescing. |
| **12-13** | **Reserved** | 2 B | `0x0000` | Reserved. |
| **14-15** | **SQID** | 2 B | `Unique` | Submission Queue Identifier. |
| **16-17** | **SQHD** | 2 B | `0x0000` | Submission Queue Head Doorbell. |
| **18-19** | **SQTP** | 2 B | `0x0000` | Submission Queue Tail Pointer. |
| **20-23** | **CDW10** | 4 B | `0x0000000E` | **Command Dword 10**: <br>Bits 07-00: **FID** = `0xE` (Timestamp).<br>Bits 31: (save bit) SV=0 or SV=1|
| **24-27** | **CDW11** | 4 B | `See Below` | **Command Dword 11**: Contains the Interrupt Coalescing parameters.<br>Bits 07-00: **Time Threshold (TIME)** in 100us units.<br>Bits 15-08: Reserved.<br>Bits 23-16: **Number of Commands (THR)** threshold.<br>Bits 31-24: Reserved. |
| **28-31** | **CDW12** | 4 B | `0x00000000` | Reserved for Set Features (FID 0xE). |
| **32-35** | **CDW13** | 4 B | `0x00000000` | Reserved. |
| **36-39** | **CDW14** | 4 B | `0x00000000` | Reserved. |
| **40-43** | **CDW15** | 4 B | `0x00000000` | Reserved. |
| **44-47** | **PRP1** | 8 B | `0x00000000` | Physical Region Page 1. **Must be 0** (no data buffer needed). |
| **48-55** | **PRP2** | 8 B | `0x00000000` | Physical Region Page 2. **Must be 0**. |
| **56-63** | **Reserved** | 8 B | `0x0000...` | Reserved. |

### Key Details for FID 0xE (Timestamp)

*   **NSID (Bytes 08-11):** Must be set to `0xFFFFFFFF` or `0x00000000` (depending on spec version, usually `0` for global features) because Interrupt Coalescing is a controller-level feature, not namespace-specific.
*   **CDW10 (Bytes 20-23):**
    - Byte 0 (Bits `07`-`00`): `0Eh` (Feature ID: Interrupt Coalescing).
    - Bytes 1-2 (Bits `23`-`08`): `00h` (Reserved).
    - Byte 3 (Bit `31`): SV Bit.
    - Set to 1 to save to non-volatile memory.
    - Set to 0 for volatile only.
    - Hex Value (Little Endian):
    - With Save: `0x8000000E` → Bytes: `0E 00 00 80`<br>
        Byte 0 = 0Eh = 14 = 000 1110<br>
        Byte 1 = 0h<br>
        Byte 2 = 0h<br>
        Byte 4 (31st bit) = 1 bit = 1000 0000 = 80<br>
    - Without Save: `0x0000000E` → Bytes: `0E 00 00 00`

![alt text](image.png)



*   **CDW11 (Bytes 24-27):** This is the only dword carrying valid data for this command.
    *   **TIME (Bits 07-00):** Time threshold in units of 100 microseconds. The controller will wait up to this time before generating an interrupt if the command threshold isn't met. Value `0` disables the time threshold.
    *   **THR (Bits 23-16):** Number of commands threshold. The controller will generate an interrupt once this many commands have completed. Value `0` disables the command count threshold.
*   **PRP1/PRP2 (Bytes 44-55):** Must be cleared to `0` because the Set Features command does not transfer data to/from memory for this feature.

### Example CDW11 Calculation
If you want an interrupt after **5 commands** OR **50 microseconds** (whichever comes first):
*   **TIME:** 50us / 100us = 0.5 -> Round to nearest valid integer (often 1 unit = 100us). Let's say `1` (100us).
*   **THR:** 5 commands.
*   **CDW11 Value:** `(5 << 16) | 1` = `0x00050001`.

All other dwords (CDW12-CDW15 reserved (0).



 ---
## nvme cli

For **Set Features, FID `0x0e`**, use NVMe Admin opcode **`0x09`** with `admin-passthru`.

FID `0x0e` is the **Timestamp** feature. It uses an **8-byte data buffer**, so you must send data with `--write`.

### 1. Create an 8-byte timestamp buffer

Timestamp is normally milliseconds since Unix epoch, little-endian, 48-bit, plus 2 bytes reserved/attributes.

```bash
python3 - <<'PY'
import time
ms = int(time.time() * 1000)

# 6-byte little-endian timestamp + 2 reserved bytes
data = ms.to_bytes(6, "little") + b"\x00\x00"

with open("/tmp/nvme_timestamp.bin", "wb") as f:
    f.write(data)

print("timestamp ms:", ms)
print("hex:", data.hex())
PY
```

### 2. Send Set Features FID `0x0e` using `admin-passthru`

```bash
sudo nvme admin-passthru /dev/nvme0 \
  --opcode=0x09 \
  --namespace-id=0 \
  --cdw10=0x0e \
  --data-len=8 \
  --write \
  --input-file=/tmp/nvme_timestamp.bin
```

Explanation:

| Option | Meaning |
|---|---|
| `--opcode=0x09` | NVMe Admin Set Features command |
| `--cdw10=0x0e` | Feature Identifier = Timestamp |
| `--data-len=8` | Timestamp feature uses 8-byte data structure |
| `--write` | Host writes data to controller |
| `--input-file=...` | File containing the 8-byte timestamp data |

### Optional: use Save bit

If you want to set the feature with the **Save** bit set, use bit 31 of CDW10:

```bash
--cdw10=0x8000000e
```

Full command:

```bash
sudo nvme admin-passthru /dev/nvme0 \
  --opcode=0x09 \
  --namespace-id=0 \
  --cdw10=0x8000000e \
  --data-len=8 \
  --write \
  --input-file=/tmp/nvme_timestamp.bin
```

Not all controllers support saving this feature.

### 3. Read it back with Get Features

Using `admin-passthru`:

```bash
sudo nvme admin-passthru /dev/nvme0 \
  --opcode=0x0a \
  --namespace-id=0 \
  --cdw10=0x0e \
  --data-len=8 \
  --read \
  --raw-binary | xxd -g1
```

Or using the normal nvme-cli command:

```bash
sudo nvme get-feature /dev/nvme0 -f 0xe -H
```

### Simpler non-passthru equivalent

Normally you can use:

```bash
sudo nvme set-feature /dev/nvme0 \
  -f 0xe \
  --data-len=8 \
  --data=/tmp/nvme_timestamp.bin
```

But the equivalent `admin-passthru` command is the `opcode=0x09`, `cdw10=0x0e`, `--write`, `--data-len=8` version shown above.