
> In NVMe, **metadata** is additional information associated with a logical block that is stored separately from (or appended to) the user data.

Think of it as:

```
Logical Block
+-------------------+----------------+
| User Data         | Metadata       |
+-------------------+----------------+
```

The metadata is **not part of the file contents** seen by applications. Instead, it contains information used by storage software, filesystems, databases, or data-protection mechanisms.

# Example

Suppose an NVMe namespace uses:

```
LBA Size = 4096 bytes
Metadata Size = 8 bytes
```

Then each logical block looks like:

```
+------------------+----------+
| 4096-byte Data   | 8-byte MD|
+------------------+----------+
```

Total:

```
4104 bytes
```

per logical block.

# Why Metadata Exists

Metadata is commonly used for:

- End-to-end data protection
- Checksums
- CRCs
- Reference tags
- Application tags
- Storage software bookkeeping

Without metadata:

```
Data only
```

With metadata:

```
Data + integrity information
```

# Protection Information (PI)

The most common use of metadata in NVMe is **Protection Information (PI)**, based on the T10 DIF/DIX standards.

Example:

```
Data Block
+--------------------+
| User Data          |
+--------------------+

Metadata
+----------+----------+----------+
| Guard    | App Tag  | Ref Tag  |
+----------+----------+----------+
```

## Guard Field

Typically:

```
16 bits
```

Contains a CRC/checksum.

Example:

```
Guard = CRC16(data)
```

The controller verifies:

```
Stored CRC == Calculated CRC
```

to detect corruption.

## Application Tag

Used by software.

Example:

```
Filesystem ID
Database ID
Application-specific value
```

## Reference Tag

Usually contains:

```
Expected LBA
```

This helps detect:

```
Wrong block written
Wrong block read
```

errors.

# Metadata Transfer Modes

NVMe supports two ways to transfer metadata.
## 1. Extended LBA

Metadata is appended to the data.

```
Logical Block

+-------------------+----------+
| User Data         | Metadata |
+-------------------+----------+
```

Example:

```
4096 bytes data
8 bytes metadata
```

appear as one 4104-byte transfer.

## 2. Separate Buffer

Metadata is transferred through a separate buffer.

```
Host

Data Buffer
     |
     +----> User Data

Metadata Buffer
     |
     +----> Metadata
```

This is controlled using:

```
MPTR
```

(Metadata Pointer)

in the NVMe command.

# Where Is Metadata Defined?

Metadata capability is described in the namespace's [[LBAF (Logical Block Address Format)]].

The Identify Namespace data structure contains:

```
LBAF
```

(Logical Block Address Format)

Each LBAF specifies:

|Field|Meaning|
|---|---|
|LBADS|Data size|
|MS|Metadata size|
|RP|Relative performance|

Example:

```
LBAF 0

LBADS = 12
MS    = 8
```

Meaning:

```
Data Size     = 2^12 = 4096 bytes
Metadata Size = 8 bytes
```

# Example Identify Namespace

```
LBAF 0

Data = 4096 B
Metadata = 8 B
```

Block layout:

```
+-------------------------+
| 4096 bytes user data    |
+-------------------------+
| 8 bytes metadata        |
+-------------------------+
```

# Metadata vs NVMe Internal Metadata

A common point of confusion:

### Host-visible Metadata

Defined by the NVMe specification.

Example:

```
Guard Tag
Reference Tag
Application Tag
```

Transferred through NVMe commands.

### Internal SSD Metadata

The SSD also maintains its own private metadata:

```
FTL mappings
Wear leveling info
Bad block tables
Garbage collection data
Journals
```

This is **not** the metadata field discussed in NVMe commands.

It is invisible to the host.

# Read Command Example

Host issues:

```
Read LBA 100
```

Controller returns:

```
User Data
+
Metadata
```

Example:

```
Data:
  4096 bytes

Metadata:
  CRC
  App Tag
  Ref Tag
```

Host verifies integrity.

# Write Command Example

Host sends:

```
4096-byte data
+
8-byte metadata
```

Controller stores both.

Later:

```
Read
```

returns the same pair.

# Visual Summary

```
NVMe Logical Block

+-----------------------------------+
| User Data (e.g. 4096 bytes)       |
+-----------------------------------+
| Metadata (e.g. 8 bytes)           |
|  - Guard Tag (CRC)                |
|  - Application Tag                |
|  - Reference Tag                  |
+-----------------------------------+
```

```
Host
  |
  | Read/Write
  v

NVMe Namespace

LBA 0
+-----------+----------+
| Data      | Metadata |
+-----------+----------+

LBA 1
+-----------+----------+
| Data      | Metadata |
+-----------+----------+
```

### Key Takeaway

> In NVMe, **metadata is host-visible information associated with each logical block**, typically used for **data integrity and protection information**. It is defined by the namespace's LBA format and may be transferred either as part of an extended LBA or through a separate metadata buffer pointed to by the command's MPTR field.

