
> An NVMe read request flows from application → filesystem → NVMe SQE → PCIe doorbell → controller DMA/FTL/NAND read → DMA to host memory → CQE completion back to the host.

![[host to nvme.png]]

This is essentially the complete NVMe datapath:

- software
- queues
- DMA
- PCIe
- controller RTL
- FTL
- NAND

all working together.


# High-Level Flow

```
Application
   ↓
Filesystem
   ↓
NVMe Driver
   ↓
Submission Queue Entry (SQE)
   ↓
PCIe Doorbell
   ↓
NVMe Controller
   ↓
FTL lookup
   ↓
NAND read
   ↓
DMA to host memory
   ↓
Completion Queue Entry (CQE)
   ↓
Interrupt/Polling
   ↓
Application receives data
```

# 🔹 Detailed Step-by-Step Process

# 1) Application Issues Read

Example:

```
read(fd, buffer, 4096)
```

Application requests:

- read 4 KB
- from some file offset

# 2) Filesystem Converts File Offset → Logical Block Address (LBA)

Filesystem determines:

```
File offset → LBA 100
Length      → 1 block
```

# 3) Block Layer Creates BIO / Request

Linux block layer builds an I/O request:

```
READ:
  SLBA = 100
  NLB  = 0  (1 block)
```

# 4) NVMe Driver Allocates Data Buffer

Host memory buffer prepared:

```
Buffer:
  Physical Address = 0x10000000
  Size = 4096 bytes
```

This buffer will receive payload data.

# 5) Driver Builds Submission Queue Entry (SQE)

64-byte command entry created.

## Example READ SQE

```
DW0:
  OPC = 0x02 (READ)
  CID = 0x1234

DW1:
  NSID = 1

DW6–DW7:
  PRP1 = 0x10000000

DW10–DW11:
  SLBA = 100

DW12:
  NLB = 0
```

# 6) Driver Places SQE into Submission Queue

Submission Queue is host memory:

```
SQ[Tail] = SQE
Tail++
```

# 7) Host Rings Submission Queue Doorbell

Host writes MMIO register:

```
SQ Tail Doorbell = new tail value
```

Over PCIe BAR memory space.

This notifies controller:

```
"New command available"
```


# 8) NVMe Controller Fetches SQE via DMA

Controller DMA engine reads SQE from host memory:

```
DMA READ:
  host SQ memory → controller
```

# 9) Command Decode

Controller parses:

```
Opcode = READ
NSID   = 1
SLBA   = 100
NLB    = 0
PRP1   = 0x10000000
```

# 10) Queue Scheduler Selects Command

Internal scheduler:

- prioritizes queues
- allocates resources

# 11) FTL Performs LBA → Physical Mapping

FTL lookup:

```
LBA 100 → NAND page P12345
```

This mapping is usually stored in:

- DRAM
- SRAM cache

# 12) NAND Read Issued

NAND controller executes flash read:

```
Read physical NAND page
```

Latency here is significant:

- tens to hundreds of microseconds

# 13) ECC Engine Corrects Errors

Controller performs:

- BCH/LDPC ECC correction

```
Raw NAND data
   ↓
ECC correction
   ↓
Validated payload
```

# 14) Controller DMA Writes Data to Host Buffer

Using PRP1:

```
DMA WRITE:
  SSD → host memory
  address = 0x10000000
```

Payload now exists in host RAM.

# 15) Controller Builds Completion Queue Entry (CQE)

Example CQE:

```
DW0:
  command-specific

DW2:
  SQHD + SQID

DW3:
  SUCCESS
  Phase Tag
```

# 16) CQE Written to Completion Queue

Controller DMA writes CQE:

```
CQ[Tail] = CQE
```

# 17) Controller Updates CQ Tail

Completion queue advanced internally.

# 18) Interrupt or Polling

Host learns completion via:

## ✔ MSI-X interrupt

or

## ✔ Polling


# 19) Driver Processes Completion

Driver:

- validates phase tag
- checks status code

```
SUCCESS
```

# 20) Driver Returns Data to Application

Application buffer now contains payload:

```
Host RAM ← data from SSD
```

Read operation completes.


# 🔹 Timeline Summary

```
Host:
  build SQE
  ring doorbell
        ↓
Controller:
  fetch SQE
  decode
  FTL lookup
  NAND read
  ECC
  DMA payload
  post CQE
        ↓
Host:
  completion
  application wakeup
```

# 🔹 Key Hardware Components Involved

| Component         | Role              |
| ----------------- | ----------------- |
| Submission Queue  | carries command   |
| Doorbell register | notify controller |
| DMA engine        | move data         |
| PRP/SGL           | locate buffer     |
| FTL               | map LBA→physical  |
| NAND controller   | read flash        |
| ECC engine        | correct errors    |
| Completion Queue  | return status     |