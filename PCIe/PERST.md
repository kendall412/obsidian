> PERST (often written PERST#, PERST_L or simply PERST) is the active-low reset signal for PCI Express devices and cards. It provides a global hardware reset that initializes PCIe endpoints and root complex functions during system power-up, card insertion, or platform-managed resets.

**PERST#** means **PCIe Reset** or **Fundamental Reset**.
The `#` means the signal is **active-low**:

```
PERST# = 0  → reset assertedPERST# = 1  → reset deasserted / released
```

### What PERST# does

PERST# resets the PCIe device/end-point hardware, such as an NVMe SSD.
For an NVMe SSD, asserting PERST# usually resets:

```
PCIe PHY
PCIe controller/MAC
LTSSM state machine
PCIe configuration logic
NVMe controller logic
internal firmware boot/reset flow
```

When PERST# is released, the device starts link training again:

```
PERST# deasserted
      |
      v
PCIe PHY starts
      |
      v
LTSSM Detect
      |
      v
Polling
      |
      v
Configuration
      |
      v
L0
      |
      v
PCIe enumeration
      |
      v
NVMe controller initialization
```

### Why PERST# exists

PERST# gives the host/platform a hardware way to put the PCIe endpoint into a known clean state before enumeration.

It is used during:

```
system power-on
warm reset
platform reset
device recovery
hot-plug/hot-reset flows
```

For M.2 NVMe SSDs, **PERST# is one of the key sideband reset signals from the host platform to the SSD.**

### PERST# timing idea

A simplified power-up sequence:

```
Power rails stable
      |
      v
Reference clock stable
      |
      v
PERST# held low
      |
      v
PERST# released high
      |
      v
Endpoint begins PCIe link training
```

Diagram:

```
Power     ___/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
Refclk    ___/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
PERST#    _________/‾‾‾‾‾‾‾‾‾‾
LTSSM              Detect → Polling → Configuration → L0
```

The host should not expect the endpoint to train or enumerate while PERST# is still asserted.

### PERST# and LTSSM

When PERST# is asserted:

```
LTSSM is reset / inactive
PHY is reset
Link is down
No TS1/TS2 exchange
No TLP traffic
No NVMe commands
```

When PERST# is deasserted:

```
LTSSM starts from Detect
```

So PERST# is earlier than normal PCIe protocol activity. It happens before the link reaches L0.

### PERST# vs PCIe Hot Reset

They are different.

|Reset type|Level|Description|
|---|---|---|
|**PERST#**|Electrical sideband signal|Physical reset pin/signal to endpoint|
|**Hot Reset**|PCIe protocol mechanism|Sent through the PCIe link using training sequences|
|**NVMe controller reset**|NVMe register mechanism|Host clears/sets `CC.EN` to reset NVMe controller|

PERST# is the most hardware-level reset of the three.

### PERST# vs NVMe controller reset

An NVMe controller reset usually means:

```
Host writes CC.EN = 0
waits for CSTS.RDY = 0
then sets CC.EN = 1
waits for CSTS.RDY = 1
```

That resets the NVMe controller function, but it does **not necessarily reset the entire PCIe physical link**.

PERST# is stronger:

```
PERST# reset
  → PCIe link reset
  → configuration state reset
  → LTSSM restarts
  → NVMe controller restarts
```

## In an NVMe SSD boot

Typical sequence:

```
1. SSD power rails come up
2. Refclk becomes stable
3. Host holds PERST# low
4. SSD remains in reset
5. Host releases PERST#
6. PCIe LTSSM starts at Detect
7. Link reaches L0
8. Host enumerates PCIe device
9. Host maps BAR0
10. Host initializes NVMe registers
```

# Common PERST# debug problems

If PERST# is wrong, the SSD may not enumerate.

Common issues:

```
PERST# never deasserts
PERST# deasserts before power/refclk are stable
PERST# pulse too short
PERST# glitches
PERST# voltage level incorrect
endpoint reset sequencing bug
```

Debug symptom examples:

```
LTSSM stuck in Detect
device not visible in lspci
NVMe SSD disappears after reboot
works after cold boot but not warm reboot
Gen4/Gen5 link unstable after reset
```

## Simple summary

> **PERST# is the PCIe hardware reset signal.** The host holds it low to reset the endpoint. When the host releases it high, the PCIe device exits reset, starts LTSSM from Detect, trains the link, reaches L0, and then can be enumerated and initialized.














---
Signal semantics:
Active-low: asserted low to hold devices in reset; deasserted (driven high) to allow normal operation.

Global effect: resets PCIe link logic, configuration space, and often device-specific state so the device appears as if power-cycled.

Typical use and timing:

Driven by platform (motherboard or system controller) or card logic (on hot-pluggable boards).

Must remain asserted until stable power rails and reference clocks (e.g., REFCLK) are present and meet specifications.

After deassertion, devices require a defined time before link training and configuration begins. The PCIe spec and platform firmware specify minimum hold times and delays; implementations commonly wait several milliseconds to tens of milliseconds.

Electrical/implementation details:

Active-low open-drain/TTL-compatible input on many devices; often pulled up on the card or system side.

For M.2, Mini PCIe, and many mezzanine connectors, PERST# is a dedicated pin. For PCIe slots on motherboards it's typically routed from the chipset or power-management controller.

In hot-swap scenarios, PERST# can be asserted automatically while slot power stabilizes, preventing spurious enumeration.

Interaction with other signals:

Complementary to power-good signals (VCC, VDD) and reference clock: PERST# should be held until power-good and clock-valid are asserted.

Software/Firmware: after PERST# deassertion, firmware/OS enumerates the device and may issue secondary resets via configuration registers if needed.

Practical guidance for designers:
Ensure PERST# is asserted during power sequencing until all required rails and clocks are stable.

Observe PCIe spec timing for deassertion-to-commence-link-training intervals.

Provide appropriate pull-up and drive strength; consider ESD and hot-plug protection for exposed connectors.

For card designers, include a PERST# input and local reset logic if the card has onboard power sequencing or needs to isolate subsystems.

Examples/typical scenarios:
Desktop motherboard: BIOS holds PERST# low until chipset power rails and REFCLK are stable; when deasserted devices begin link training and the OS enumerates them.

Hot-pluggable PCIe slot: controller asserts PERST# while slot power is applied; once power-good and safety checks pass, PERST# is released to enable the card.

Embedded system: a system controller toggles PERST# to recover a malfunctioning PCIe endpoint without cycling system power.


Relevant standards and references:
PCI Express Base Specification defines PERST# behavior, timing parameters, and interaction with power and clock signals.

Platform-specific documents (M.2, Mini PCIe, card-edge slot specifications) show pin assignments and recommended sequencing.

Summary

PERST is the mandatory hardware reset line for PCIe devices used to guarantee correct power-up, prevent premature enumeration, and support safe hot-plug/reset operations. Proper sequencing and timing of PERST# relative to power and clock are critical for reliable PCIe operation.