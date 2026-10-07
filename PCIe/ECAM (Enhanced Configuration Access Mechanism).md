#ecam
The **Enhanced Configuration Access Mechanism (ECAM)** is the PCIe mechanism that lets software access PCIe Configuration Space using ordinary **memory-mapped addresses**. The key idea is:

> Software performs a memory access to a specially reserved ECAM address, and the Root Complex/host bridge translates that access into a PCIe **Configuration Read or Configuration Write Request**.

The PCIe Base Specification defines ECAM as a flat memory-mapped address space, with portions of the address encoding the Bus, Device, Function, and configuration-register offset. [Intel](https://www.intel.com/content/dam/support/us/en/programmable/support-resources/fpga-wiki/asset03/pci-express-base-r2.1.pdf?utm_source=chatgpt.com)

### 1. ECAM gives each Function 4 KiB of address space
Every PCIe Function may have up to **4 KiB of Configuration Space**.

ECAM maps that Configuration Space into the processor's physical address space:
```
System Physical Address Space

          ┌─────────────────────────┐
          │ Normal RAM              │
          │                         │
          ├─────────────────────────┤
ECAM Base │ PCIe ECAM region        │
   ──────►│                         │
          │ Bus/Device/Function     │
          │ Configuration Spaces    │
          │                         │
          ├─────────────────────────┤
          │ Other MMIO              │
          └─────────────────────────┘
```

The ECAM region's base address is platform-defined and provided to the OS by firmware. [Intel](https://www.intel.com/content/dam/support/us/en/programmable/support-resources/fpga-wiki/asset03/pci-express-base-r2.1.pdf?utm_source=chatgpt.com)

### 2. The ECAM address encodes the BDF
The important part is how the address is constructed.

For the common full 8-bit Bus Number mapping:
```
ECAM offset

27            20 19        15 14      12 11                0
┌───────────────┬────────────┬──────────┬────────────────────┐
│  Bus Number   │   Device   │ Function │ Register Offset    │
│    8 bits     │   5 bits   │  3 bits │      12 bits       │
└───────────────┴────────────┴──────────┴────────────────────┘
```

More precisely, the specification divides the low 12 bits further:

```
Bits 27:20    Bus Number
Bits 19:15    Device Number
Bits 14:12    Function Number
Bits 11:8     Extended Register Number
Bits  7:2     Register Number
Bits  1:0     Byte / byte-enable information
```

This mapping is also reflected directly in Linux's PCI ECAM definitions: Bus is shifted by 20 bits and Device/Function occupy the region beginning at bit 12. [GitHub](https://github.com/torvalds/linux/blob/master/include/linux/pci-ecam.h?utm_source=chatgpt.com)

Therefore, a useful formula is:

Address = ECAMBase + (Bus << 20) + (Device << 15) + (Function << 12) + Offset

### 3. Example: accessing an NVMe device
Suppose your NVMe SSD has:
```
BDF = 03:00.0

Bus      = 0x03
Device   = 0x00
Function = 0
```

And suppose the ECAM base is:
```
ECAM Base = 0xE0000000
```

You want to read configuration offset:
```
0x000
```

which contains the Vendor ID and Device ID.

Calculate:
```
Address =
    0xE0000000
  + (0x03 << 20)
  + (0x00 << 15)
  + (0x00 << 12)
  + 0x000
```

So:
```
Bus 3:

0x03 << 20
       =
0x00300000
```

Therefore:
```
0xE0000000
+0x00300000
───────────
 0xE0300000
```

The CPU/software accesses:
```
0xE0300000
```


### 4. What happens when the CPU reads that address?
This is the important part. The CPU isn't directly sending a Configuration Request TLP.

It performs what looks like a memory read:
```
CPU

Read physical address
0xE0300000
       │
       ▼
Host Bridge / Root Complex
```

The host hardware recognizes:
```
0xE0300000
```

as being inside its ECAM range.

It then decodes the address:
```
0xE0300000
       │
       ▼

Bus      = 03
Device   = 00
Function = 0
Offset   = 000
```

and generates the appropriate PCIe Configuration Request. ECAM implementations translate accesses to the ECAM memory aperture into PCIe configuration reads/writes. [AMD Documentation](https://docs.amd.com/r/en-US/pg055-axi-bridge-pcie/Enhanced-Configuration-Access?utm_source=chatgpt.com)

So the complete flow is:
```
Software
   │
   │ read 0xE0300000
   ▼
CPU
   │
   │ physical memory access
   ▼
Host Bridge / Root Complex
   │
   │ recognizes ECAM address
   │
   │ decodes:
   │
   │ Bus      = 03
   │ Device   = 00
   │ Function = 0
   │ Register = 000
   ▼
Creates PCIe
Configuration Read Request
   │
   ▼
Root Port
   │
   ▼
PCIe fabric
   │
   ▼
NVMe Endpoint
```


### 5. The PCIe Configuration Request is generated
The Root Complex now generates something conceptually like:
```
Configuration Read Request

Type       = CfgRd0 / CfgRd1
Bus        = 03
Device     = 00
Function   = 0
Register   = 000
Requester  = Root Complex
Tag        = xx
```

This is now an actual PCIe **TLP**, not a CPU memory transaction.

That's an important distinction:
```
CPU side                        PCIe side

Memory Read
0xE0300000
      │
      ▼
┌─────────────────┐
│  Root Complex   │
│                 │
│ ECAM translation│
└────────┬────────┘
         │
         ▼
Configuration Read TLP
BDF = 03:00.0
Offset = 0x000
```

ECAM is therefore essentially the **software-facing mechanism that causes the Root Complex to generate the PCIe Configuration Request**.


### 6. The Endpoint becomes the Completer
This connects directly to your previous question about [[completer]].

The NVMe Endpoint receives the Configuration Read:
```
Root Complex                         NVMe SSD

 Requester                           Completer
     │                                   │
     │       CfgRd0                      │
     │ BDF = 03:00.0                     │
     │ Offset = 0x000                    │
     │──────────────────────────────────►│
     │                                   │
     │                    Read Config    │
     │                    Space 0x000    │
     │                                   │
     │            CplD                   │
     │◄──────────────────────────────────│
```

The Endpoint returns a **Completion with Data (CplD)**.

Suppose its Configuration Space contains:

```
Offset 0x000

31                    16 15                     0
┌───────────────────────┬────────────────────────┐
│      Device ID        │       Vendor ID        │
└───────────────────────┴────────────────────────┘
```

The returned data ultimately propagates back through the Root Complex to the original CPU load.

### 7. Configuration Writes work similarly
Suppose software wants to write configuration offset `0x004`.

Software calculates:
```
ECAM Base
+ (Bus << 20)
+ (Device << 15)
+ (Function << 12)
+ 0x004
```

and performs a memory write.

Then:
```
Software
    │
    │ Memory Write to ECAM address
    ▼
CPU
    │
    ▼
Root Complex
    │
    │ recognizes ECAM
    ▼
Configuration Write Request
    │
    ▼
PCIe Endpoint
    │
    │ updates Configuration Space
    ▼
Completion
```

Configuration Writes are **Non-Posted**, so unlike an ordinary PCIe Memory Write, they require a Completion. [AMD Documentation](https://docs.amd.com/r/en-US/pg055-axi-bridge-pcie/Enhanced-Configuration-Access?utm_source=chatgpt.com)

### Why 4 KiB per Function?
It should be 4KiB aligned.
Notice that Function occupies bits `[14:12]`. That means bits `[11:0]` remain for the configuration-space offset:

2^12=4096 bytes

So each Function receives exactly:
```
Function 0
0x000 - 0xFFF
      =
4096 bytes
      =
4 KiB
```

Then the next Function starts `0x1000` bytes later:
```
Function 0    +0x0000 – +0x0FFF
Function 1    +0x1000 – +0x1FFF
Function 2    +0x2000 – +0x2FFF
...
Function 7    +0x7000 – +0x7FFF
```

Similarly:
```
8 Functions × 4 KiB
        =
32 KiB per Device
```

and:

```
32 Devices × 32 KiB
        =
1 MiB per Bus
```

which is why the Bus Number starts at address bit 20. The full ECAM layout naturally follows from those powers of two. [GitHub](https://github.com/torvalds/linux/blob/master/include/linux/pci-ecam.h?utm_source=chatgpt.com)

### Putting everything together
For an NVMe SSD at `03:00.0`:
```
Software wants:
"Read PCIe configuration offset 0x000"
              │
              ▼
Calculate ECAM address

ECAM_BASE
+ (03 << 20)
+ (00 << 15)
+ (0  << 12)
+ 0x000
              │
              ▼
CPU performs memory read
              │
              ▼
Root Complex recognizes ECAM range
              │
              ▼
Decodes BDF + Register
              │
              ▼
Creates Configuration Request TLP
              │
              ▼
         PCIe Fabric
              │
              ▼
         NVMe Endpoint
          (Completer)
              │
              ▼
Reads its Configuration Space
              │
              ▼
            CplD
              │
              ▼
         Root Complex
              │
              ▼
CPU receives requested value
```

So the distinction to remember is:

**ECAM is not itself a PCIe TLP.** It is a **memory-mapped host access mechanism**. A CPU load/store to the ECAM aperture causes the Root Complex to generate the corresponding **PCIe Configuration Request TLP**.

And this is also why ECAM supports the full **4 KiB PCIe Configuration Space**, including the Extended Configuration Space above `0xFF`, whereas the older PCI-compatible configuration mechanism is fundamentally based around the original 256-byte configuration model. [Intel](https://www.intel.com/content/dam/support/us/en/programmable/support-resources/fpga-wiki/asset03/pci-express-base-r2.1.pdf?utm_source=chatgpt.com)

