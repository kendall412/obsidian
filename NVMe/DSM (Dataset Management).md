**Dataset Management (DSM)** is an NVMe **NVM command** that lets the host give the SSD hints about ranges of logical blocks. The most common DSM operation is **Deallocate**, which is NVMe’s equivalent of **TRIM/UNMAP**.

In simple terms:

> DSM tells the SSD: “These LBAs are no longer needed,” or provides hints about how certain data ranges are used.

## Command type

DSM is part of the **NVM Command Set**, not the Admin command set.

```text
Opcode: 0x09
Command: Dataset Management
```

It is sent to a **namespace**, for example:

```text
/dev/nvme0n1
```

not just the controller:

```text
/dev/nvme0
```

## Main use: Deallocate / TRIM

The most important DSM attribute is:

```text
AD = Attribute - Deallocate
```

When the host sets the **Deallocate** bit, it tells the SSD that the specified LBA ranges no longer contain useful data.

Example meaning:

```text
LBAs 100000 through 199999 are no longer needed.
```

The SSD can then:

- mark those blocks as invalid
- avoid copying them during garbage collection
- improve write performance
- reduce write amplification
- free internal flash resources
- potentially improve endurance

This is what filesystems issue when files are deleted or when `fstrim` is run.

## Why DSM exists

NAND flash cannot overwrite data in-place. The SSD has to erase larger flash blocks before rewriting them.

If the host deletes a file but does not tell the SSD, the SSD may think the old LBAs still contain valid data. That causes unnecessary copying during garbage collection.

DSM Deallocate helps by telling the SSD:

> “You do not need to preserve this data anymore.”

## DSM command structure

A DSM command includes one or more **range descriptors**. Each descriptor describes a range of LBAs.

A range descriptor typically includes:

- **Starting LBA**
- **Number of logical blocks**
- **Context attributes / hints**

The command can describe multiple ranges in a single DSM command.

Conceptually:

```text
Range 1: Start LBA = 1000, Length = 500 blocks
Range 2: Start LBA = 9000, Length = 100 blocks
Range 3: Start LBA = 20000, Length = 2000 blocks
```

## DSM attributes

DSM can carry different attributes/hints, depending on controller support and specification version. Commonly discussed attributes include:

### 1. Deallocate

```text
AD = 1
```

The specified logical blocks are no longer required.

This is the TRIM-like behavior.

### 2. Integral Dataset for Read / Write hints

These are hints that a range may be read or written together as a dataset.

They are less commonly used in normal Linux desktop/server workflows.

## Read behavior after Deallocate

After a DSM Deallocate/TRIM, what happens if you read the same LBAs?

That depends on namespace settings and controller behavior.

The drive may return:

- zeroes
- old data
- indeterminate data

NVMe Identify Namespace reports this behavior using fields such as:

```text
DLFEAT
```

For security, **DSM Deallocate is not the same as secure erase**. Do not rely on TRIM/DSM to securely destroy data unless the device explicitly guarantees that behavior.

## Linux examples

### Check whether DSM is supported

```bash
sudo nvme id-ctrl -H /dev/nvme0
```

Look for Dataset Management support in the command support fields.

You can also inspect namespace information:

```bash
sudo nvme id-ns -H /dev/nvme0n1
```

Relevant fields may include:

```text
dlfeat
```

### Use filesystem TRIM

Most users do not call DSM directly. They use filesystem tools.

Run TRIM on a mounted filesystem:

```bash
sudo fstrim -v /
```

This causes the OS to issue deallocate/discard commands to the drive for unused filesystem space.

### Block discard

Discard a block device range, or entire device if used carefully:

```bash
sudo blkdiscard /dev/nvme0n1
```

Be careful: `blkdiscard` can make data inaccessible.

### Direct nvme-cli DSM command

`nvme-cli` has DSM support:

```bash
sudo nvme dsm /dev/nvme0n1 --ad --slbs=<starting-lba> --blocks=<number-of-blocks>
```

Example:

```bash
sudo nvme dsm /dev/nvme0n1 --ad --slbs=100000 --blocks=4096
```

This tells the SSD that 4096 logical blocks starting at LBA 100000 are deallocated.

Syntax can vary slightly by `nvme-cli` version, so check:

```bash
nvme dsm --help
```

## DSM vs Write Zeroes

DSM Deallocate and Write Zeroes are related but different.

### DSM Deallocate

Means:

```text
I no longer need this data.
```

The SSD may internally free or invalidate it. Future reads may or may not return zeroes depending on device behavior.

### Write Zeroes

Means:

```text
Make this range read back as zeroes.
```

This is stronger from a data semantics perspective, but may have different performance behavior.

## DSM vs Format / Sanitize

DSM is not a full-drive erase operation.

| Operation | Purpose |
|---|---|
| DSM Deallocate | Tell SSD specific ranges are unused |
| Write Zeroes | Make ranges read as zero |
| Format NVM | Reinitialize namespace/format |
| Sanitize | Destroy user data across media/cache according to sanitize action |
| Crypto Erase | Destroy encryption key so data is unrecoverable |

## Simple summary

**Dataset Management** is an NVMe command that lets the host describe LBA ranges and provide usage hints. Its most common use is **Deallocate**, the NVMe version of **TRIM**, where the host tells the SSD that certain logical blocks no longer contain useful data. This helps the SSD improve garbage collection, performance, and endurance, but it is not the same as secure erase.

Example: **Dataset Management** is an NVMe **I/O command** with opcode:

```text
Opcode = 09h
```

A common use is **Deallocate / TRIM** a range of LBAs.

---

## Example: Deallocate 128 LBAs starting at LBA 0x1000

### Command summary

```text
Command:        Dataset Management
Opcode:         09h
Namespace ID:   1
Action:         Deallocate
Starting LBA:   0x1000
Number of LBAs: 128
```

---

## NVMe command fields

### Submission Queue Entry

| Field | Value | Meaning |
|---|---:|---|
| `OPC` | `09h` | Dataset Management |
| `NSID` | `1` | Namespace ID |
| `PRP1/SGL` | address of range list | Points to DSM range descriptors |
| `CDW10` | `0` | Number of ranges, zero-based: `0` means 1 range |
| `CDW11` | `00000004h` | Attribute: AD bit set = Deallocate |

For Dataset Management:

```text
CDW11 bit 2 = AD, Attribute Deallocate
```

So:

```text
CDW11 = 1 << 2 = 0x00000004
```

---

## Dataset Management range descriptor

The data buffer contains one or more 16-byte range descriptors.

For one range:

| Bytes | Field | Example value |
|---:|---|---:|
| `0–3` | Context Attributes | `00000000h` |
| `4–7` | Number of Logical Blocks | `00000080h` |
| `8–15` | Starting LBA | `0000000000001000h` |

Little-endian byte layout:

```text
00 00 00 00   // Context Attributes
80 00 00 00   // Number of LBAs = 128
00 10 00 00 00 00 00 00   // Starting LBA = 0x1000
```

---

## nvme-cli example

On Linux, the equivalent command is typically:

```bash
nvme dsm /dev/nvme0n1 \
  --ad \
  --slbs=4096 \
  --blocks=128
```

Where:

```text
--ad           request deallocate
--slbs=4096    starting LBA = 0x1000
--blocks=128   number of logical blocks
```

---

## Minimal pseudo-command

```c
cmd.opcode = 0x09;          // Dataset Management
cmd.nsid   = 1;

cmd.prp1   = dma_addr_of_dsm_range_list;

cmd.cdw10  = 0;             // one range
cmd.cdw11  = 0x4;           // AD = Deallocate

range[0].cattr = 0;
range[0].nlb   = 128;
range[0].slba  = 0x1000;
```

This submits an NVMe I/O command asking the controller to deallocate, or TRIM, LBAs `0x1000` through `0x107F`.