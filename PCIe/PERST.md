PERST (often written PERST#, PERST_L or simply PERST) is the active-low reset signal for PCI Express devices and cards. It provides a global hardware reset that initializes PCIe endpoints and root complex functions during system power-up, card insertion, or platform-managed resets.

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