
> For an NVMe controller, the registers are located in the MMIO space pointed to by BAR0. The exact register map depends on the NVMe specification revision, but the standard register layout looks like this:

# NVMe Controller Register Map

![[controller_BAR.png]]


```
BAR0 Base Address
(e.g. 0x80000000)
        |
        v

Offset
======  ==================================================

0x0000  CAP      Controller Capabilities             (64b)
        +----------------------------------------+
        | MQES | CQR | AMS | TO | DSTRD | ...    |
        +----------------------------------------+

0x0008  VS       Version                            (32b)
        +----------------------------------------+
        | MJR | MNR | TER                        |
        +----------------------------------------+

0x000C  INTMS    Interrupt Mask Set                (32b)

0x0010  INTMC    Interrupt Mask Clear              (32b)

0x0014  CC       Controller Configuration          (32b)
        +----------------------------------------+
        | EN | CSS | MPS | AMS | SHN | IOSQES... |
        +----------------------------------------+

0x0018  Reserved

0x001C  CSTS     Controller Status                 (32b)
        +----------------------------------------+
        | RDY | CFS | SHST | NSSRO | PP          |
        +----------------------------------------+

0x0020  NSSR     NVM Subsystem Reset               (32b)

0x0024  AQA      Admin Queue Attributes            (32b)
        +----------------------------------------+
        | ACQS | ASQS                           |
        +----------------------------------------+

0x0028  ASQ      Admin Submission Queue Base Addr (64b)

0x0030  ACQ      Admin Completion Queue Base Addr (64b)

0x0038  CMBLOC   Controller Memory Buffer Location

0x003C  CMBSZ    Controller Memory Buffer Size

0x0040  BPINFO   Boot Partition Information

0x0048  BPRSEL   Boot Partition Read Select

0x0050  BPMBL    Boot Partition Memory Buffer Loc

0x0058  CMBMSC   Controller Memory Buffer Memory Space Control

0x0060  CMBSTS   Controller Memory Buffer Status

0x0068  CMBEBS   Controller Memory Buffer Elasticity Buffer Size

0x0070  CMBSWTP  Controller Memory Buffer Sustained Write Throughput

0x0078  NSSD     NVM Subsystem Shutdown

0x0080  CRTO     Controller Ready Timeouts

0x0088+ Vendor-Specific Registers (optional)
```

# Doorbell Register Area

After the controller registers comes the doorbell region.

Typically:

```
0x1000
```

and above.

```
0x1000  SQ0 Tail Doorbell
0x1004  CQ0 Head Doorbell

0x1008  SQ1 Tail Doorbell
0x100C  CQ1 Head Doorbell

0x1010  SQ2 Tail Doorbell
0x1014  CQ2 Head Doorbell

...
```

General form:

```
Doorbell[n]

Queue 0
+-------------------+
| SQ0 Tail DB       |
+-------------------+
| CQ0 Head DB       |
+-------------------+

Queue 1
+-------------------+
| SQ1 Tail DB       |
+-------------------+
| CQ1 Head DB       |
+-------------------+

Queue 2
+-------------------+
| SQ2 Tail DB       |
+-------------------+
| CQ2 Head DB       |
+-------------------+
```

# NVMe Register Space Hierarchy

```
PCIe Device
│
├── PCIe Configuration Space
│    │
│    └── BAR0
│         |
│         v
│
└── NVMe MMIO Register Space

     0x0000 CAP
     0x0008 VS
     0x000C INTMS
     0x0010 INTMC
     0x0014 CC
     0x001C CSTS
     0x0020 NSSR
     0x0024 AQA
     0x0028 ASQ
     0x0030 ACQ
     ...
     0x1000 Doorbells
     ...
```

# Registers Used During Initialization

The host typically touches these registers in order:

```
1. Read CAP
       |
       v
2. Read VS
       |
       v
3. Program AQA
       |
       v
4. Program ASQ
       |
       v
5. Program ACQ
       |
       v
6. Program CC.EN=1
       |
       v
7. Poll CSTS.RDY
       |
       v
8. Create Admin Queues
       |
       v
9. Ring Doorbells
```

Diagrammatically:

```
Host CPU
   |
   | Read CAP
   v
+---------+
| CAP     |
+---------+

   |
   | Configure Queues
   v
+---------+
| AQA     |
+---------+
| ASQ     |
+---------+
| ACQ     |
+---------+

   |
   | Enable Controller
   v
+---------+
| CC.EN=1 |
+---------+

   |
   | Poll
   v
+---------+
| CSTS    |
+---------+

   |
   | Submit Commands
   v
+-------------------+
| SQ Doorbells      |
+-------------------+
| CQ Doorbells      |
+-------------------+
```

For day-to-day NVMe firmware and driver development, the most important registers are:

- **CAP** (Capabilities)
- **VS** (Version)
- **CC** (Controller Configuration)
- **CSTS** (Controller Status)
- **AQA** (Admin Queue Attributes)
- **ASQ** (Admin Submission Queue Base Address)
- **ACQ** (Admin Completion Queue Base Address)
- **Doorbell Registers** (all queue activity flows through these)
