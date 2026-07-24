
> The **Controller Configuration (CC)** register is one of the most important registers in NVMe. It is a **memory-mapped controller register** that the host writes to configure and enable the NVMe controller. The CC register is located in the NVMe controller's PCIe BAR space and is accessed through [[MMIO (Memory-Mapped IO)]].

> (NVMe Base Spec Rev. 2.3: 3.1.4.5) This property modifies settings for the controller. A host shall set the Arbitration Mechanism Selected (CC.AMS), the Memory Page Size (CC.MPS), and the I/O Command Set Selected (CC.CSS) to valid values prior to enabling the controller by setting CC.EN to ‘1’. Attempting to create an I/O queue before initializing the I/O Completion Queue Entry Size (CC.IOCQES) and the I/O Submission Queue Entry Size (CC.IOSQES) shall cause a controller to abort a Create I/O Completion Queue command or a Create I/O Submission Queue command with a status code of Invalid Queue Size.

# Purpose of CC Register

The host uses the CC register to:

1. Enable or disable the NVMe controller
2. Select I/O command set
3. Configure memory page size
4. Configure arbitration mechanism
5. Configure shutdown behavior
6. Configure Submission Queue and Completion Queue entry sizes

A controller cannot process commands until the host properly configures CC and enables it.

# Initialization Sequence

During controller initialization:

```
1. PCIe enumeration
2. Host maps BAR0
3. Host reads CAP register
4. Host programs:
      AQA
      ASQ
      ACQ
5. Host programs CC register
6. Host sets CC.EN = 1
7. Controller becomes ready
8. CSTS.RDY = 1
```

# CC Register Layout

The CC register is a 32-bit register.

```
Controller Configuration (CC)

31                                      0
+--+--+-----+-----+----+----+---+------+
|EN|RSV|CSS  |MPS  |AMS |SHN |IOSQES|
+--+--+-----+-----+----+----+---+------+
|               IOCQES                |
+--------------------------------------+
```

More precisely:

| Bits  | Field    |
| ----- | -------- |
| 0     | EN       |
| 3:1   | Reserved |
| 6:4   | CSS      |
| 10:7  | MPS      |
| 13:11 | AMS      |
| 15:14 | SHN      |
| 19:16 | IOSQES   |
| 23:20 | IOCQES   |
| 31:24 | Reserved |
# 1. EN — Enable

Bit:

```
CC[0]
```

Values:

```
0 = Controller disabled
1 = Controller enabled
```

Example:

```
CC.EN = 1
```

causes controller startup.

## Disable Procedure

Host clears:

```
CC.EN = 0
```

Controller eventually reports:

```
CSTS.RDY = 0
```

Only after RDY clears may the host reconfigure queues.

# 2. CSS — [[Command Set]] Selected

Bits:

```
CC[6:4]
```

Selects the command set.

Examples:

|Value|Meaning|
|---|---|
|000b|NVM Command Set|
|110b|All supported I/O command sets|
|Others|Vendor or future use|

Most SSDs use:

```
CSS = 000b
```

for standard NVM commands.

# 3. MPS — Memory Page Size

Bits:

```
CC[10:7]
```

Defines host memory page size used by NVMe structures.

Formula:

```
Page Size = 2^(12 + MPS)
```

Examples:

|MPS|Page Size|
|---|---|
|0|4 KB|
|1|8 KB|
|2|16 KB|
|3|32 KB|

Most systems:

```
MPS = 0
```

(4 KB pages)


# 4. AMS — Arbitration Mechanism Selected

Bits:

```
CC[13:11]
```

Determines how Submission Queues are serviced.

Examples:

|Value|Method|
|---|---|
|000b|Round Robin|
|001b|Weighted Round Robin with Urgent|
|Others|Vendor specific|

Most systems:

```
AMS = 000b
```

# 5. SHN — Shutdown Notification

Bits:

```
CC[15:14]
```

Used before system power-off.

Values:

|Value|Meaning|
|---|---|
|00b|Normal operation|
|01b|Normal shutdown|
|10b|Abrupt shutdown|
|11b|Reserved|

Example:

```
CC.SHN = 01b
```

Host requests a clean shutdown.

Controller later reports:

```
CSTS.SHST = Shutdown Complete
```


# 6. IOSQES — I/O Submission Queue Entry Size

Bits:

```
CC[19:16]
```

Power-of-two encoding.

Formula:

```
SQ Entry Size = 2^(IOSQES)
```

Standard NVMe SQ entry:

```
64 bytes
```

Therefore:

```
IOSQES = 6
```

because:

```
2^6 = 64
```


# 7. IOCQES — I/O Completion Queue Entry Size

Bits:

```
CC[23:20]
```

Formula:

```
CQ Entry Size = 2^(IOCQES)
```

Standard completion entry:

```
16 bytes
```

Therefore:

```
IOCQES = 4
```

because:

```
2^4 = 16
```


# Typical CC Value

A common configuration:

```
EN      = 1
CSS     = 000b
MPS     = 0
AMS     = 0
SHN     = 0
IOSQES  = 6
IOCQES  = 4
```

Binary:

```
0100 0110 0000 0001b
```

Hex:

```
0x00460001
```


# Relationship Between CC and CSTS

The host writes CC; the controller reports status through the Controller Status (CSTS) register.

```
Host                     Controller
----                     ----------
CC.EN = 1  ---------->
                    initialize
                    queues
                    admin engine
                    firmware

               CSTS.RDY = 1

Host may now send commands
```

Similarly:

```
CC.EN = 0  ---------->
                    stop processing
                    flush state

               CSTS.RDY = 0
```

# Key Point

The **Controller Configuration (CC)** register is the host's primary control register for bringing an NVMe controller online. Before any Admin or I/O commands can be processed, the host must:

1. Configure **ASQ**, **ACQ**, and **AQA**
2. Program **CC** (page size, queue entry sizes, command set)
3. Set **CC.EN = 1**
4. Wait for **CSTS.RDY = 1**

Only then is the NVMe controller operational and ready to accept commands.

