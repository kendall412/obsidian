# NVMe Admin Command Set – Opcode Cheat Sheet

### Core Structure

- **Opcode (OPC)** = 1 byte in **DWORD 0**
- Executed via **Admin Submission Queue**
- Applies to **controller + namespaces (management only)**

### Initialization & Queue Management

|Opcode|Command|Purpose|
|---|---|---|
|`0x00`|Delete I/O Submission Queue|Remove an SQ|
|`0x01`|Create I/O Submission Queue|Create SQ|
|`0x04`|Delete I/O Completion Queue|Remove CQ|
|`0x05`|Create I/O Completion Queue|Create CQ|

### Identification & Capabilites
|Opcode|Command|Purpose|
|---|---|---|
|`0x06`|Identify|Get controller/namespace info|

### Feature Management
|Opcode|Command|Purpose|
|---|---|---|
|`0x09`|Set Features|Configure controller behavior|
|`0x0A`|Get Features|Read current settings|
### Log & Monitoring
|Opcode|Command|Purpose|
|---|---|---|
|`0x02`|Get Log Page|Retrieve SMART, error logs, stats|
### Firmware Management
|Opcode|Command|Purpose|
|---|---|---|
|`0x10`|Firmware Commit|Activate/update firmware|
|`0x11`|Firmware Image Download|Transfer firmware image|
### Namespace Management
|Opcode|Command|Purpose|
|---|---|---|
|`0x0D`|Namespace Management|Create/Delete namespaces|
|`0x15`|Namespace Attachment|Attach/Detach namespaces|
### Format & Sanitize
| Opcode | Command    | Purpose                   |
| ------ | ---------- | ------------------------- |
| `0x80` | Format NVM | Format a namespace        |
| `0x84` | Sanitize   | Secure erase / purge data |

### Security
| Opcode | Command          | Purpose                     |
| ------ | ---------------- | --------------------------- |
| `0x81` | Security Send    | Send security protocol data |
| `0x82` | Security Receive | Receive security data       |

### Misc & Advanced
|Opcode|Command|Purpose|
|---|---|---|
|`0x08`|Abort|Abort a running command|
|`0x0C`|Async Event Request|Get async notifications|
|`0x14`|Device Self-test|Run diagnostics|
|`0x18`|Keep Alive|Prevent controller timeout|
|`0x19`|Directive Send|Send directives|
|`0x1A`|Directive Receive|Receive directives|
|`0x1C`|Virtualization Mgmt|Manage SR-IOV / virtualization|
|`0x1D`|NVMe-MI Send|Management interface send|
|`0x1E`|NVMe-MI Receive|Management interface receive|

### Command Set & Modern NVMe (2.0+)
|Opcode|Command|Purpose|
|---|---|---|
|`0x1F`|Identify Namespace Descriptor List|Extended namespace info|
|`0x20`|Identify I/O Command Set|Discover supported command sets|


### Notes

- **Admin commands = control plane only** (no user data reads/writes)
- **All NVMe devices must support Admin command set**
- Some opcodes are:
    - Optional
    - Version-dependent (NVMe 1.3 vs 2.0+)
- `0xC0–0xFF` → **Vendor-specific**

### Ultra-Short Summary

- **0x00–0x05** → Queue management
- **0x06** → Identify
- **0x09–0x0A** → Features
- **0x02** → Logs
- **0x10–0x11** → Firmware
- **0x0D–0x15** → Namespaces
- **0x80+** → Format / Security / Sanitize


