
> The **PCIe Transaction Layer** is the top PCIe protocol layer. Its job is to convert software/device requests into **Transaction Layer Packets (TLPs)**.

```
+----------------------------+
| Software / Driver / NVMe   |
+----------------------------+
| PCIe Transaction Layer     |  <-- creates and consumes TLPs
+----------------------------+
| PCIe Data Link Layer       |  <-- ACK/NAK, replay, LCRC
+----------------------------+
| PCIe Physical Layer        |  <-- sends bits on lanes
+----------------------------+
```

## Main job

The Transaction Layer handles operations like:

- Memory read
- Memory write
- Configuration Read (registers)
- Configuration Write (registers)
- Completion
- Message
- Atomic operations


For NVMe, it is what turns host actions like:

1. write NVMe [[doorbell registers]]
2. read [[Controller Capabilities (CAP)]] register
3. fetch command from host memory
4. DMA data to host memory

into PCIe packet transactions.

## TLP basic structure

A PCIe TLP looks conceptually like this:

```
+-------------------------+
| TLP Header              |
+-------------------------+
| Data Payload (optional) |
+-------------------------+
| ECRC optional           |
+-------------------------+
```

The header tells the receiver:

```
What type of transaction?
Where is it going?
How many bytes?
Who sent it?
Does it require a completion?
```

## Important TLP types

### 1. Memory Write TLP

Used to write data to memory or [[MMIO (Memory-Mapped IO)]] registers.

Example: host rings an NVMe submission queue doorbell.

```
Host CPU writes SQ Tail Doorbell
        |
        v
PCIe Memory Write TLP
        |
        v
NVMe Controller BAR0 doorbell register
```

Memory Write usually does **not** require a Completion TLP.

### 2. Memory Read TLP

Used to request data.

Example: host reads NVMe CAP register.

```
Host sends Memory Read TLP
        |
        v
NVMe SSD receives request
        |
        v
SSD sends Completion with Data
```

Memory Read requires a Completion.

### 3. Completion TLP

Used to return the result of a request.

Example:

```
Host: Memory Read CAP register
SSD:  Completion with CAP register value
```

The Completion uses fields like **Requester ID** and **Tag** so the requester can match the response to the original request.

### 4. Configuration TLPs

Used during PCIe enumeration.

Example:

```
Read Vendor ID
Read Device ID
Discover BAR size
Program BAR address
Enable bus mastering
```

These happen before the OS can fully use the NVMe device.

### 5. Message TLPs

Used for PCIe events and control messages.

Examples:

```
MSI/MSI-X interrupt messages
Power management messages
Error reporting messages
Hot-plug messages
```

An NVMe completion interrupt through MSI-X is delivered as a PCIe Message/MSI-style write mechanism depending on implementation.

## Important Transaction Layer fields

A TLP header includes fields such as:

| Field         | Purpose                                                 |
| ------------- | ------------------------------------------------------- |
| Fmt/Type      | Identifies Memory Read, Memory Write, Completion, etc.  |
| Length        | Payload size in DWORDs                                  |
| Requester ID  | Bus/device/function of requester                        |
| Completer ID  | Bus/device/function of completer                        |
| Tag           | Matches requests with completions                       |
| Address       | Target memory/MMIO address                              |
| Byte Enables  | Which bytes are valid                                   |
| Attributes    | Ordering, snooping, relaxed ordering, ID-based ordering |
| Traffic Class | QoS/priority class                                      |

## Posted vs Non-Posted vs Completion

PCIe transactions are grouped into three major categories.

### Posted transaction

A posted request does not require a completion.

Example:

```
Memory WriteMessage
```

The sender sends it and does not wait for a response.

```
Host ---> Memory Write TLP ---> SSD
```

### Non-posted transaction

A non-posted request requires a completion.

Example:

```
Memory Read
Configuration Read
Configuration Write
```

```
Host ---> Memory Read TLP ---> SSD
Host <--- Completion TLP <---- SSD
```

### Completion

A response to a non-posted request.

```
Completion without Data
Completion with Data
Completion with Error Status
```

## Flow control credits

The Transaction Layer also works with PCIe flow control. Before sending TLPs, a transmitter must know the receiver has buffer space.

Credits exist for categories such as:

```
Posted Header
Posted Data
Non-Posted Header
Non-Posted Data
Completion Header
Completion Data
```

Example:

```
If receiver has no posted data credits,
sender cannot send more Memory Write payloads.
```

This prevents buffer overflow.

## Ordering rules

The Transaction Layer enforces PCIe ordering rules. This matters because PCIe allows some performance optimizations, but certain operations must remain ordered.

Example NVMe doorbell case:

```
1. Host writes command into Submission Queue memory
2. Host writes SQ Tail Doorbell
```

**The SSD must not see the doorbell before the command data is visible in memory.** Drivers use memory barriers and PCIe ordering rules to ensure correct behavior.

## Address translation role

The **Transaction Layer uses PCIe addresses, not NVMe LBAs.**

Example:

```
NVMe LBA = storage address inside SSD
PCIe Address = host memory or MMIO address
```

For a Read command:

```
NVMe command says:
  Read LBA 1000

PRP/SGL says:
  Put data at host memory address 0x12340000

PCIe Transaction Layer generates:
  Memory Write TLPs to 0x12340000
```

## NVMe example: host reads CAP register

```
1. CPU reads BAR0 + 0x0000
2. Root Complex creates Memory Read TLP
3. TLP goes to NVMe SSD
4. SSD Transaction Layer receives TLP
5. SSD returns Completion with Data
6. Host receives CAP value
```

Conceptually:

```
Host                         NVMe SSD
----                         --------
Memory Read TLP -----------> CAP register
Completion w/Data <--------- CAP value
```

## NVMe example: host submits a command

```
1. Host writes 64-byte NVMe command into Submission Queue in host RAM
2. Host writes SQ Tail Doorbell in NVMe BAR0
3. Doorbell write becomes PCIe Memory Write TLP
4. SSD Controller sees doorbell update
5. SSD Controller DMA-fetches command from host memory using Memory Read TLP
6. Host returns Completion with command data
7. SSD Controller executes NVMe command
```

Important point:

```
NVMe command itself is not automatically the same thing as a TLP.
```

> The NVMe command is a 64-byte structure in host memory. PCIe TLPs are used to move/register-access around it.

## Transaction Layer vs Data Link Layer

|Layer|Responsibility|
|---|---|
|Transaction Layer|Creates Memory Read/Write/Completion/Config/Message TLPs|
|Data Link Layer|Adds sequence numbers, LCRC, ACK/NAK, replay|
|Physical Layer|Serializes and transmits bits over lanes|

The Transaction Layer decides **what transaction to perform**.

The Data Link Layer ensures the packet is delivered reliably across one link.

