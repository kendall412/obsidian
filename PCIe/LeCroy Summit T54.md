
Below is a practical setup guide for using a **Teledyne LeCroy Summit T54 PCIe Gen5 Protocol Analyzer** with the interposers you are most likely to encounter in NVMe/SSD validation: **M.2, U.2/U.3, EDSFF, CEM/slot, and older Gen4 interposers**.

The T54 is a **PCIe Gen5 x4 analyzer**, supporting PCIe traffic through **32 GT/s per lane**. For widths greater than x4, different analyzer configurations or multiple analyzers may be required depending on the interposer and use case. ([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/pdf/cross-sync-phy-datasheet.pdf?utm_source=chatgpt.com "CrossSync PHY, Cross-layer Analysis Datasheet"))

## 1. Basic T54 bench architecture

Conceptually, every setup looks like this:

```text
                     Control connection
                  USB / Ethernet
                        │
                        ▼
                ┌────────────────┐
                │ Summit T54     │
                │ Protocol       │
                │ Analyzer       │
                └───────┬────────┘
                        │
                  High-speed
                 analyzer cable
                        │
                        ▼
HOST ─────────► [ INTERPOSER ] ─────────► DUT
                    │
                    │
                 +12 V DC
```

The interposer is inserted **electrically between the host and DUT**. It passes the PCIe link through while tapping both directions for the analyzer:

```text
Upstream traffic:
DUT ─────────────────────────────► Host

Downstream traffic:
Host ────────────────────────────► DUT

                 │
                 │ passive/active tap
                 ▼
             Summit T54
```

The analyzer therefore sees:

- LTSSM/link training
    
- TS1/TS2 ordered sets
    
- DLLPs
    
- TLPs
    
- NVMe Admin commands
    
- NVMe I/O commands
    
- Completion traffic
    
- PCIe errors/retries
    
- reset and power-on sequences
    

Teledyne LeCroy specifically recommends having the analyzer recording **before the system under test is powered on** when you need the complete power-on trace. ([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/manuals/pciegen5x8interposerquickstartguide.pdf?utm_source=chatgpt.com "1 | Introduction | ® PCI Express 5.0 Slot Interposer User Manual and Quick Start Guide None"))

---

# 2. Common setup sequence

Regardless of interposer type, use approximately this sequence.

### Hardware OFF initially

```text
Host/SUT       OFF
DUT            not powered
Interposer     connect first
T54            can initially be off
```

Connect:

```text
PC
 │ USB/Ethernet
 ▼
T54
 │
 │ analyzer cable
 ▼
Interposer
 │
 ├──────── Host
 │
 └──────── DUT
```

Then:

```text
1. Configure interposer switches
2. Install DUT
3. Install host-side connection
4. Connect interposer → T54 cable
5. Connect interposer 12-V supply
6. Connect T54 → control PC
7. Power on T54
8. Open PCIe Protocol Analysis software
9. Verify analyzer is detected
10. Start recording
11. Power on host/SUT
12. Allow PCIe enumeration
13. Stop recording when desired
14. Inspect CATC trace
```

This order is particularly important when you want to capture:

```text
PERST#
   ↓
Detect
   ↓
Polling
   ↓
Configuration
   ↓
Recovery / Equalization
   ↓
L0
   ↓
PCIe Enumeration
   ↓
NVMe Initialization
   ↓
Admin Commands
```

Teledyne LeCroy's CEM setup specifically calls for **Analyzer ON → start recording → Host ON** to obtain a power-on trace. ([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/manuals/pciegen5x8interposerquickstartguide.pdf?utm_source=chatgpt.com "1 | Introduction | ® PCI Express 5.0 Slot Interposer User Manual and Quick Start Guide None"))

---

# 3. Software setup

Install **Teledyne LeCroy PCIe Protocol Analysis Software** on the control PC.

As of July 2026, Teledyne LeCroy lists the current T54 package as:

```text
PCIe Analysis Software: v13.36
LinkExpert:             v5.36
Package release:        2026.10
```

Both online and full offline installers are available. ([Teledyne LeCroy](https://www.teledynelecroy.com/support/softwaredownload/psgdocuments.aspx?mseries=594&standardid=3&utm_source=chatgpt.com "Teledyne LeCroy - Analysis Software for PCI Express"))

Connect the PC to the T54 using:

```text
USB
or
Ethernet
```

Ethernet is often more convenient for a permanent validation bench.

---

# 4. M.2 NVMe SSD setup

For NVMe SSD validation, this will probably be one of your most common arrangements.

A current Gen5 M.2 interposer family is the **PE222UIA series**. It supports M-Key NVMe devices and PCIe Gen1 through Gen5, with x1/x2/x4 operation. ([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/pdf/m2pcie5-interposer-datasheet.pdf?utm_source=chatgpt.com "PCI Express® 5.0"))

Typical topology:

```text
                        USB/Ethernet
                             │
                             ▼
                      ┌──────────────┐
                      │ Summit T54   │
                      └──────┬───────┘
                             │
                    PE027 / PE028 cable
                             │
                             ▼
        ┌────────────────────────────────┐
        │ PCIe Gen5 M.2 Interposer       │
        │                                │
Host ───┤ Host paddle                    │
        │                                │
        │                 M.2 SSD DUT ───┤
        └────────────────────────────────┘
                       │
                      12 V
```

The M.2 interposer supports common SSD lengths including:

```text
2230
2242
2260
2280
22110
```

and certain current designs accommodate widths up to 30 mm. ([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/pdf/m2pcie5-interposer-datasheet.pdf?utm_source=chatgpt.com "PCI Express® 5.0"))

### Step 1 — Clock setting

Look for the interposer clock-selection DIP switch, typically identified as something similar to:

```text
SW2
```

For normal testing, use:

```text
HOST_CLK
```

Teledyne LeCroy explicitly recommends HOST_CLK as the default unless you intend to use an external reference clock. ([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/pdf/m2pcie5-interposer-datasheet.pdf?utm_source=chatgpt.com "PCI Express® 5.0"))

Conceptually:

```text
Host REFCLK
      │
      ▼
Interposer
      │
      ├──── SSD
      │
      └──── Analyzer clock reference
```

Do not select external clock mode unless your test topology actually provides the appropriate external reference.

### Step 2 — Install SSD

Insert the M.2 SSD into the DUT socket.

Example:

```text
        SSD
   ┌───────────────┐
   │ NVMe M.2      │
   └───────────────┘
          ╲
           ╲ connector
            ▼
      ┌───────────────┐
      │ Interposer    │
      └───────────────┘
```

Secure the SSD using the appropriate standoff.

Avoid running a Gen5 SSD loosely connected. Mechanical movement can create signal-integrity problems that masquerade as PCIe link issues.

### Step 3 — Install host paddle

The interposer uses a paddle/card connection into the host M.2 connector.

```text
Motherboard M.2 socket
        │
        ▼
    Paddle PCB
        │
        │ cable / interposer routing
        ▼
M.2 Interposer
        │
        ▼
      SSD DUT
```

Make sure the correct keyed paddle is used.

### Step 4 — Connect interposer to T54

The Gen5 M.2 documentation identifies these cables:

```text
PE027UCA-X
PE028UCA-X
```

The exact cable configuration depends on analyzer mode. Teledyne LeCroy specifies the straight/Y-cable selection according to whether MultiPort operation is being used. ([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/pdf/m2pcie5-interposer-datasheet.pdf?utm_source=chatgpt.com "PCI Express® 5.0"))

Do not substitute a similar-looking Gen4 cable. High-speed analyzer cables are generation/configuration specific.

### Step 5 — Power interposer

Connect the supplied:

```text
12-V DC adapter
```

The interposer electronics require their own power; this is not simply the M.2 SSD's power source.

### Step 6 — Start capture

Use:

```text
T54 ON
    ↓
PCIe Analysis Software
    ↓
Start Recording
    ↓
Power Host
```

Then you should see something like:

```text
PCIe Electrical Activity
        ↓
TS1 / TS2
        ↓
Gen1 initial training
        ↓
Speed Change
        ↓
Gen5 Equalization
        ↓
L0
        ↓
Configuration Read/Writes
        ↓
NVMe Controller Initialization
```

---

# 5. U.2 / U.3 NVMe setup

The Gen5 U.2/U.3 interposer is particularly useful for enterprise SSD validation.

It supports:

```text
PCIe Gen1    2.5 GT/s
PCIe Gen2    5 GT/s
PCIe Gen3    8 GT/s
PCIe Gen4    16 GT/s
PCIe Gen5    32 GT/s
```

and can operate as:

```text
single x4 port
or
dual x2 ports
```

It also exposes/handles sideband signals including:

```text
PERST#
WAKE#
CLKREQ#
SMBus
  SMBCLK
  SMBDAT
```

([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/manuals/pciegen5x4u2u3interposerquickstart.pdf?utm_source=chatgpt.com "PCI Express® 5.0 U.2/U.3 Interposer User Manual and Quick Start Guide"))

Typical setup:

```text
                   Control PC
                       │
                       ▼
                  Summit T54
                       │
                Analyzer cable
                       │
                       ▼
            ┌───────────────────┐
Host ──────►│ U.2/U.3 Interposer│──────► SSD
Backplane   └───────────────────┘        U.2/U.3
                       │
                      12 V
```

For an enterprise server:

```text
Server backplane
      │
      ▼
U.2/U.3 connector
      │
      ▼
Interposer
      │
      ▼
Enterprise NVMe SSD
```

The analyzer cable connects separately:

```text
Interposer
     │
     └────────► Summit T54
```

### Important U.2/U.3 issue

Do not treat U.2 and U.3 as electrically identical merely because the connector looks similar.

U.3 introduces tri-mode pin assignment/routing considerations. Make sure:

```text
Host mode
+
Interposer configuration
+
DUT type
```

are compatible.

For a PCIe/NVMe validation environment, verify that the selected port is actually operating as PCIe rather than SATA/SAS/tri-mode routing through some other controller.

---

# 6. EDSFF SSD setup

For newer datacenter SSD validation you may encounter:

```text
E1.S
E1.L
E3.S
E3.L / E3 variants
```

The Gen5 EDSFF interposer family supports devices such as:

```text
E1.S x4
E1.S x8
E1.L
E3
```

depending on the specific interposer. Teledyne LeCroy's Gen5 EDSFF platform supports PCIe speeds through **32 GT/s** and may support link widths as high as x16 depending on model. ([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/manuals/gen5edsff-interposer-quick-start.pdf?utm_source=chatgpt.com "PCI Express® Gen4 x4 EDSFF Interposer User Manual and Quick Start Guide"))

Conceptually:

```text
                   Summit T54
                       │
                analyzer cable
                       │
                       ▼
        ┌────────────────────────┐
Host ──►│ EDSFF Interposer       │──► E1.S SSD
        │                        │
        └────────────────────────┘
                     │
                    12 V
```

EDSFF interposers often use interchangeable mounting hardware because the physical dimensions are substantially different between:

```text
E1.S thin
E1.S thick
E1.L
E3
```

Install the correct bracket before inserting the DUT.

### For an E1.S x4 SSD

A typical NVMe validation topology would be:

```text
Host E1.S slot
      │
      ▼
EDSFF E1.S interposer
      │
      ├────► Summit T54
      │
      ▼
E1.S NVMe DUT
```

For x4 devices, a T54 is well suited because the analyzer itself supports up to x4 Gen5 traffic.

### For x8/x16 EDSFF

Be careful here.

A single T54 is fundamentally a **Gen5 x4 analyzer**. An x8/x16 EDSFF interposer may physically support more lanes than one T54 can capture.

You may therefore need:

```text
multiple analyzer channels
or
T516 / M5x / other analyzer configuration
```

depending on what portion of the link must be captured.

The interposer's maximum lane width and the analyzer's capture width are separate specifications.

---

# 7. PCIe CEM / slot interposer setup

This is used when the DUT is a standard add-in PCIe card.

Examples:

```text
NIC
GPU
NVMe adapter
FPGA card
accelerator
storage controller
```

Topology:

```text
                     Summit T54
                         │
                  analyzer cable
                         │
                         ▼
          ┌─────────────────────────┐
Host ────►│ PCIe CEM Interposer     │
          │                         │
          │           PCIe DUT ─────┤
          └─────────────────────────┘
                         │
                        12 V
```

The interposer plugs into the motherboard:

```text
Motherboard
PCIe slot
   ▲
   │
┌──┴────────────────┐
│ CEM Interposer    │
├───────────────────┤
│ DUT connector     │
└───────▲───────────┘
        │
     PCIe card
```

Teledyne LeCroy's Gen5 CEM instructions specify:

1. Insert interposer into host PCIe CEM slot.
    
2. Insert DUT into the interposer.
    
3. Supply 12-V DC power to the interposer.
    
4. Connect interposer to T54 using the selected analyzer cable.
    
5. Connect analyzer to control PC.
    
6. Start the PCIe analysis application.
    
7. Power the analyzer.
    
8. Start recording.
    
9. Power the host. ([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/manuals/pciegen5x8interposerquickstartguide.pdf?utm_source=chatgpt.com "1 | Introduction | ® PCI Express 5.0 Slot Interposer User Manual and Quick Start Guide None"))
    

The Gen5 slot interposer supports negotiation to the common active width depending on the DUT and interposer lane-width configuration. ([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/manuals/pciegen5x8interposerquickstartguide.pdf?utm_source=chatgpt.com "1 | Introduction | ® PCI Express 5.0 Slot Interposer User Manual and Quick Start Guide None"))

---

# 8. T54 with older Gen4 interposers

The T54 isn't limited to Gen5 interposers.

Teledyne LeCroy lists compatibility with several Gen4 systems, including:

```text
PE180UIA-X     Gen4 x1 slot
PE181UIA-X     Gen4 x4 slot
PE186UIA-X     Gen4 M.2
PE188UIA-X     Gen4 U.2/U.3
PE200UIA-X     Gen4 EDSFF
PE163UIA-X     Gen4 OCuLink
PE182UIA-X     Gen4 x8 slot
```

([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/manuals/pciegen5_summitt54_quickstart.pdf?utm_source=chatgpt.com "
Summit T54
PCI Express 5.0 Protocol Analyzer
Q"))

For example:

```text
                  PE020UCA-X
T54 Gen4 PHY ────────────────── Gen4 x4 Interposer
                                    │
                       Host ────────┼────── DUT
```

Teledyne LeCroy specifically documents a T54 with Gen4 PHY connected to a **PE181UIA-X** using a **PE020UCA-X Gen4 x4 straight cable**. ([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/manuals/pciegen5_protocolanalyzer_usermanual.pdf?utm_source=chatgpt.com "Summit T5 PCI Express Gen5 User Manual"))

---

# 9. Gen3/Gen2 interposers

The T54 can also work with older generation probe systems using adapter cables.

Teledyne LeCroy lists:

```text
Gen2 / Gen3 ≤ x4
       │
       ▼
PE026UCA-X adapter cable
       │
       ▼
T54
```

and for certain wider configurations:

```text
Gen2 / Gen3 ≤ x8
       │
       ▼
PE016UCA-X
       │
       ▼
T54 Expanded Mode configuration
```

([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/manuals/pciegen5_summitt54_quickstart.pdf?utm_source=chatgpt.com "
Summit T54
PCI Express 5.0 Protocol Analyzer
Q"))

This is useful in a validation lab where older interposer inventories are still being used.

---

# 10. Interposer clock configuration

Clock setup is one of the most important parts of PCIe analyzer troubleshooting.

Depending on the platform you may encounter:

```text
Common Clock
Separate Refclk Independent SSC (SRIS)
Separate Refclk No SSC (SRNS)
```

For a normal PC/server design, start with:

```text
HOST_CLK
```

On an M.2 interposer, for example:

```text
HOST REFCLK
      │
      ├────────► SSD
      │
      └────────► interposer/analyzer
```

If you incorrectly configure the interposer for external clock mode, you can get:

```text
No link
or
Detect ↔ Polling loop
or
Gen5 equalization failures
or
unstable trace decoding
```

Do not change clock mode just because the link is failing. First identify the platform's actual PCIe clock architecture.

---

# 11. Sideband signals

For NVMe testing, don't focus only on Tx/Rx lanes.

Interposers may expose:

```text
PERST#
CLKREQ#
WAKE#
SMBCLK
SMBDAT
REFCLK
power rails
```

These are extremely useful during validation.

For example, if the SSD doesn't enumerate:

```text
Check:

12 V / 3.3 V
    ↓
REFCLK present?
    ↓
PERST# asserted?
    ↓
PERST# released?
    ↓
PCIe Detect?
    ↓
TS1/TS2?
    ↓
Link reaches L0?
```

Sometimes what looks like:

> "SSD PCIe training failure"

is actually:

```text
PERST# never released
```

or:

```text
REFCLK missing
```

rather than a high-speed lane problem.

---

# 12. Link-width setting

Some interposers have a physical switch or control that lets you select active width.

Examples:

```text
x1
x2
x4
x8
x16
```

For NVMe M.2:

```text
Host = x4
SSD  = x4
Interposer = x4

→ expected x4
```

If the interposer is set to x2:

```text
Host = x4 capable
SSD  = x4 capable
Interposer active = x2

→ link may negotiate x2
```

This is useful for validation.

For example:

```text
Test 1: Gen5 x4
Test 2: Gen5 x2
Test 3: Gen5 x1
Test 4: Gen4 x4
Test 5: Gen3 x4
```

You can use this to validate width/speed fallback behavior.

---

# 13. Power-on trace setup

For SSD validation, I recommend capturing at least one full power-on trace.

Configure:

```text
Recording mode:
Start immediately

Trigger:
None initially

Buffer:
Large enough for enumeration
```

Then:

```text
Start recording
      ↓
Power on system
      ↓
PERST# sequence
      ↓
PCIe LTSSM
      ↓
L0
      ↓
PCI configuration
      ↓
NVMe initialization
```

A typical NVMe sequence should eventually show activity such as:

```text
PCIe Enumeration

Configuration Read
Configuration Write

BAR allocation

Host MMIO
    ↓
NVMe CAP read
NVMe VS read
NVMe CC programming
AQA
ASQ
ACQ
CC.EN = 1
    ↓
CSTS.RDY = 1
    ↓
Identify
Get Log Page
Set Features
Create I/O CQ
Create I/O SQ
```

That makes the T54 particularly powerful because you can correlate:

```text
PCIe link establishment
        +
PCIe configuration
        +
NVMe initialization
```

in one trace.

---

# 14. Good physical bench layout

For Gen5, cable and mechanical discipline matter.

Try to arrange:

```text
              Control PC
                  │
                  │ Ethernet
                  ▼
          ┌──────────────┐
          │ Summit T54   │
          └──────┬───────┘
                 │
          short analyzer cable
                 │
                 ▼
        ┌─────────────────┐
        │ Interposer      │
        └───────┬─────────┘
                │
              DUT
                │
              Host
```

Avoid:

```text
Analyzer cable twisted
        +
interposer hanging unsupported
        +
SSD not screwed down
        +
power cable pulling on PCB
```

At **32 GT/s**, marginal physical setups can cause link-training failures.

Teledyne LeCroy specifically warns that slot interposers should be mechanically secured to the chassis to avoid damage. ([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/manuals/pciegen5x8interposerquickstartguide.pdf?utm_source=chatgpt.com "1 | Introduction | ® PCI Express 5.0 Slot Interposer User Manual and Quick Start Guide None"))

---

# 15. Troubleshooting: analyzer sees no link

Use this order.

### First: does the host see the SSD?

Linux:

```bash
lspci
```

Then:

```bash
lspci -vv
```

For NVMe:

```bash
nvme list
```

If the host doesn't see it either, the problem isn't just analyzer decoding.

Check:

```text
DUT power
↓
Interposer power
↓
PERST#
↓
REFCLK
↓
Host connector
↓
Interposer switches
↓
Analyzer cable
↓
PCIe lane configuration
```

---

# 16. Link trains only to Gen4

Suppose:

```text
Host = Gen5 capable
SSD = Gen5 capable
T54/interposer = Gen5

but

Link = Gen4
```

Possible causes include:

```text
Signal integrity
Interposer/cable configuration
Gen5 equalization failure
BIOS maximum speed setting
DUT firmware
Host setting
Incorrect clock configuration
Retimer topology
Lane mapping
```

Use the T54 to look for:

```text
Recovery.Equalization
```

and repeated:

```text
Gen5 attempt
      ↓
equalization failure
      ↓
fallback
      ↓
Gen4 L0
```

This is one of the most useful analyzer scenarios in SSD validation.

---

# 17. Link never reaches L0

Trace may look like:

```text
Detect
 ↓
Polling.Active
 ↓
Polling.Configuration
 ↓
Recovery
 ↓
Detect
 ↓
Polling...
```

Investigate:

```text
REFCLK
PERST#
Lane polarity
Lane reversal
EQ coefficients
Presets
Receiver detection
Signal integrity
Width setting
Interposer configuration
```

---

# 18. CrossSync PHY setup

Some Gen5 LeCroy interposers support **CrossSync PHY**.

This lets you correlate:

```text
Protocol Analyzer
        +
Oscilloscope
```

For example:

```text
T54 trace:
Recovery.Equalization Phase 2
             │
             │ timestamp correlated
             ▼
Oscilloscope:
Lane 0 eye / waveform
```

Topology:

```text
                     Summit T54
                         │
                         │ protocol
                         ▼
Host ───────────── Interposer ───────────── DUT
                       │
                       │ high-speed probe
                       ▼
                LeCroy Oscilloscope
```

CrossSync PHY requires compatible:

```text
oscilloscope
protocol analyzer
interposer
software/licensing
```

The T54 is among the analyzers supported by Teledyne LeCroy's CrossSync PHY system. ([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/manuals/crossync-phy-pcie-im-eng.pdf?utm_source=chatgpt.com "CrossSync PHY PCIe Instruction Manual"))

This is especially useful for:

```text
Gen5 equalization failure
receiver margining
link instability
electrical/protocol correlation
```

---

# 19. Quick reference by SSD form factor

|DUT|Typical interposer|Host connection|Maximum typical NVMe width|T54 suitability|
|---|---|---|--:|---|
|M.2|Gen5 M.2 M-Key|M.2 paddle|x4|Excellent|
|U.2|Gen5 U.2/U.3|U.2 backplane|x4|Excellent|
|U.3|Gen5 U.2/U.3|U.3 backplane|x4|Excellent|
|E1.S|Gen5 EDSFF|EDSFF backplane|x4/x8 depending DUT|Excellent for x4|
|E1.L|Gen5 EDSFF|EDSFF backplane|x4/x8|Depends on width|
|E3|Gen5 EDSFF|EDSFF backplane|up to x16 designs|Multiple/wider analyzer may be needed|
|PCIe AIC|Gen5 CEM|PCIe slot|x1–x16|Excellent through captured x4; wider links need appropriate configuration|

---

# 20. T54 lab checklist

Before powering anything, I would use this checklist:

```text
□ Correct interposer for DUT
□ Interposer supports desired PCIe generation
□ Correct analyzer cable
□ Correct cable orientation
□ DUT securely installed
□ Host connection secure
□ Interposer powered
□ HOST_CLK / clock mode correct
□ Lane width configured correctly
□ T54 detected by software
□ Firmware/software versions compatible
□ Recording started before DUT power-on
```

Then after power-up:

```text
□ Electrical activity detected
□ TS1/TS2 visible
□ Link speed correct
□ Link width correct
□ LTSSM reaches L0
□ PCIe enumeration visible
□ NVMe initialization visible
□ No unexpected malformed TLP / DLLP errors
```

The key topology to remember for SSD validation is simply:

```text
                    ┌──────────────┐
                    │ Summit T54   │
                    └──────▲───────┘
                           │
                    analyzer cable
                           │
                           │
HOST  ◄────────────── INTERPOSER ──────────────► SSD
                           │
                           │
                        12-V power
```

The **host and SSD still communicate directly through the interposer**. The T54 is the observer: it captures both directions of PCIe traffic without becoming the NVMe host or controller.

![Image](https://assets.lcry.net/images/mcio-cable-interposer.png)

![Image](https://assets.lcry.net/images/gen6_cem_interposer.png)

![Image](https://pt.teledynelecroy.com/images/pcie4-oculink.png)

For an NVMe validation bench, the three setups I would learn first are **T54 + M.2**, **T54 + U.2/U.3**, and **T54 + E1.S**, since together they cover a large fraction of client and datacenter SSD validation work. ([Teledyne LeCroy](https://cdn.teledynelecroy.com/files/pdf/m2pcie5-interposer-datasheet.pdf?utm_source=chatgpt.com "PCI Express® 5.0"))