#nvme-cli 

In `nvme-cli`, **`admin-passthru`** is a command that lets you send a **raw NVMe Admin Command** directly to an NVMe controller.

It is called “passthru” because `nvme-cli` does not interpret the command much; it mostly passes your specified opcode, namespace ID, command dwords, and optional data buffer directly to the NVMe device/driver.

Example form:

```bash
nvme admin-passthru /dev/nvme0 \
  --opcode=0x06 \
  --data-len=4096 \
  --read
```

That sends an admin command with opcode `0x06`, which is commonly **Identify**, and reads 4096 bytes of data back.

A more explicit example:

```bash
nvme admin-passthru /dev/nvme0 \
  --opcode=0x06 \
  --namespace-id=1 \
  --cdw10=1 \
  --data-len=4096 \
  --read \
  --raw-binary
```

This is roughly equivalent to issuing an NVMe Identify command.

Common options include:

```text
--opcode / -o        NVMe admin command opcode
--namespace-id / -n  Namespace ID
--cdw2 ... --cdw15   Command dword fields
--data-len / -l      Data buffer length
--read / -r          Data is read from the device
--write / -w         Data is written to the device
--input-file / -i    File containing data to send
--raw-binary / -b    Print raw binary output
```

For example, in NVMe, an admin command has fields like:

```text
Opcode
NSID
CDW10
CDW11
CDW12
CDW13
CDW14
CDW15
Data buffer
```

`admin-passthru` lets you manually fill those fields.

It is useful for:

- Sending standard NVMe admin commands manually
- Testing vendor-specific admin commands
- Debugging firmware/device behavior
- Accessing features not directly exposed by `nvme-cli`
- Sending commands from an NVMe specification document

There is also `io-passthru`, which is similar but sends **I/O commands** instead of **Admin commands**.

Important warning: `admin-passthru` can be dangerous. Some admin commands can format the drive, change firmware, delete namespaces, sanitize data, or alter controller state. You should only use it if you know exactly what opcode and command dwords mean.

For example:

```bash
nvme admin-passthru /dev/nvme0 --opcode=0x84 ...
```

could be a vendor-specific command depending on the device. Vendor-specific admin commands are not portable and require the vendor’s documentation.

`nvme admin-passthru` in **nvme-cli** does not have a fixed subcommand list. It lets you send an arbitrary **NVMe Admin Command opcode** to the controller.

Basic form:

```bash
nvme admin-passthru /dev/nvme0 \
  --opcode=<admin-opcode> \
  --namespace-id=<nsid> \
  --cdw10=<value> --cdw11=<value> ... \
  --data-len=<bytes> \
  --read | --write
```

## OPCODE

You can see your installed tool’s options with:

```bash
nvme admin-passthru --help
```

Common NVMe Admin opcodes include:

| Opcode | Admin command |
|---:|---|
| `0x00` | Delete I/O Submission Queue |
| `0x01` | Create I/O Submission Queue |
| `0x02` | Get Log Page |
| `0x04` | Delete I/O Completion Queue |
| `0x05` | Create I/O Completion Queue |
| `0x06` | Identify |
| `0x08` | Abort |
| `0x09` | Set Features |
| `0x0a` | Get Features |
| `0x0c` | Asynchronous Event Request |
| `0x0d` | Namespace Management |
| `0x10` | Firmware Commit |
| `0x11` | Firmware Image Download |
| `0x14` | Device Self-test |
| `0x15` | Namespace Attachment |
| `0x18` | Keep Alive |
| `0x19` | Directive Send |
| `0x1a` | Directive Receive |
| `0x1c` | Virtualization Management |
| `0x1d` | NVMe-MI Send |
| `0x1e` | NVMe-MI Receive |
| `0x20` | Capacity Management |
| `0x21` | Lockdown |
| `0x24` | Doorbell Buffer Config |
| `0x80` | Format NVM |
| `0x81` | Security Send |
| `0x82` | Security Receive |
| `0x84` | Sanitize |
| `0x86` | Get LBA Status |
| `0xc0`–`0xff` | Vendor-specific admin commands |

Not every controller supports every optional command. Check supported optional admin commands with:

```bash
nvme id-ctrl -H /dev/nvme0
```

Look for fields such as `oacs`, which reports Optional Admin Command Support.

Example: send an Identify Controller command using passthrough:

```bash
nvme admin-passthru /dev/nvme0 \
  --opcode=0x06 \
  --cdw10=1 \
  --data-len=4096 \
  --read
```

But normally you would use the built-in wrapper instead:

```bash
nvme id-ctrl /dev/nvme0
```

Use `admin-passthru` mainly for testing, vendor-specific commands, or commands not directly exposed by nvme-cli.