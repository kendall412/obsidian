
> The **PCIe Device Control Register** is a register in the **PCI Express Capability Structure** that allows software (BIOS, OS, or device driver) to control certain behaviors of a PCIe device. It is part of the PCIe Configuration Space.

Inside the PCIe Capability Structure:

```
PCI Configuration Space
│
├── PCIe Capability Structure
│   ├── PCIe Capabilities Register
│   ├── Device Capabilities Register
│   ├── Device Control Register   <-- Here
│   ├── Device Status Register
│   ├── Link Capabilities Register
│   ├── Link Control Register
│   └── ...
```

The Device Control Register is a **16-bit register**.
## Purpose

The register controls:

- Error reporting
- Maximum Payload Size (MPS)
- Relaxed Ordering
- No Snoop transactions
- Extended Tag support
- Phantom Functions

These settings affect how the device exchanges Transaction Layer Packets (TLPs) over PCIe.

## Register Layout

Typical PCIe Device Control Register:

```
15                     0
+--+--+--+--+--+--+--+--+
|MPS|RO|NS|ET|PF|ERR ...|
+--+--+--+--+--+--+--+--+
```

More detailed:

|Bits|Field|
|---|---|
|0|Correctable Error Reporting Enable|
|1|Non-Fatal Error Reporting Enable|
|2|Fatal Error Reporting Enable|
|3|Unsupported Request Reporting Enable|
|4|Enable Relaxed Ordering|
|5-7|Max Payload Size (MPS)|
|8|Extended Tag Field Enable|
|9|Phantom Functions Enable|
|10|Auxiliary Power PM Enable|
|11|No Snoop Enable|
|12-14|Max Read Request Size (MRRS)|
|15|Initiate Function Level Reset (optional in newer PCIe versions)|

## 1. Maximum Payload Size (MPS)

Bits:

```
[7:5]
```

Determines the largest payload the device may place in a TLP.

Possible values:

| Encoding | Payload |
| -------- | ------- |
| 000      | 128 B   |
| 001      | 256 B   |
| 010      | 512 B   |
| 011      | 1024 B  |
| 100      | 2048 B  |
| 101      | 4096 B  |

Example:

```
MPS = 256 bytes
```

A DMA write of 1024 bytes becomes:

```
256 + 256 + 256 + 256
```

4 PCIe packets.

## 2. Maximum Read Request Size (MRRS)

Bits:

```
[14:12]
```

Controls the largest Memory Read Request the device may issue.

Values:

| Encoding | Size   |
| -------- | ------ |
| 000      | 128 B  |
| 001      | 256 B  |
| 010      | 512 B  |
| 011      | 1024 B |
| 100      | 2048 B |
| 101      | 4096 B |

Example:

```
MRRS = 512 B
```

The device can request:

```
Memory Read 512 B
```

in one PCIe read request.

## 3. Relaxed Ordering Enable

Bit:

```
4
```

Allows transactions to complete out of order when permitted.

### Disabled

```
A → B → C
```

must complete in order.

### Enabled

```
A → B → C
```

could complete:

```
B → C → A
```

if performance benefits.

Useful for high-performance devices such as NVMe SSDs.

## 4. Extended Tag Enable

Bit:

```
8
```

PCIe uses tags to match read completions to read requests.

### Standard

```
32 tags
```

### Extended

```
256 tags
```

This allows many more outstanding reads.

Critical for:

- NVMe SSDs
- High-speed NICs
- GPUs

## 5. No Snoop Enable

Bit:

```
11
```

Allows the device to mark transactions as:

```
No Snoop
```

Meaning:

> Cache coherency checks may be bypassed.

Can improve performance on some platforms.

## 6. Error Reporting Enables

Bits:

```
0-3
```

Control reporting of:

### Correctable Errors

Example:

- Recoverable CRC error

### Non-Fatal Errors

Example:

- Malformed TLP that does not bring down the link

### Fatal Errors

Example:

- Serious protocol violation

### Unsupported Request Errors

Example:

```
Memory Read to invalid address
```

# NVMe Example

An NVMe SSD driver often programs:

```
MPS  = 256 B
MRRS = 512 B or 1024 B
Extended Tags = Enabled
Relaxed Ordering = Enabled
```

Why?

Because NVMe devices generate:

- Large DMA transfers
- Many outstanding commands
- Heavy PCIe traffic

These settings maximize throughput.

# Reading the Register

Linux (bash):

```
lspci -vv
```

Example:

```
DevCtl:
    Report errors: Correctable+ Non-Fatal+ Fatal+
    RlxdOrd+
    ExtTag+
    NoSnoop-
    MaxPayload 256 bytes
    MaxReadReq 512 bytes
```

This output comes directly from the PCIe Device Control Register.

## Relationship to NVMe

During PCIe enumeration:

1. BIOS/OS discovers the NVMe controller.
2. Reads PCIe Capability Structure.
3. Reads Device Capabilities Register.
4. Programs Device Control Register:
    - MPS
    - MRRS
    - Error reporting
    - Extended tags
5. NVMe driver begins controller initialization.

So the Device Control Register is one of the key PCIe registers configured **before NVMe queue creation and command processing begin**.

