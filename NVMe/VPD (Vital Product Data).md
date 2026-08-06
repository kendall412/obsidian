# VPD (Vital Product Data)

It is a standardized mechanism defined in the PCIe specification that allows a device (like an NVMe SSD, network card, or GPU) to store and expose critical identification and configuration information to the host system (OS, BIOS, or management software).

Think of VPD as a digital nameplate or a specification sheet embedded directly inside the device's firmware.

1. What Kind of Data is Stored in VPD?

    VPD data is typically read-only (though some fields can be writable by manufacturers) and includes:

    - Part Number (PN): The specific model identifier.
    - Serial Number (SN): The unique identifier for that specific unit.
    - Manufacturer Name: e.g., "SK hynix", "Samsung", "Intel".
    - Hardware/Firmware Revision: Version numbers for tracking updates.
    - Asset Tag: A custom field often used by enterprises for inventory tracking.
    - Power Consumption Limits: Critical for server chassis to calculate total power budget.
    - PCIe Link Capabilities: Specific speed or lane width constraints.

2. How VPD Works in PCIe/NVMe

    - Storage Location: The data is stored in a small, non-volatile memory area (usually EEPROM or Flash) on the device itself, accessible via the PCIe configuration space.
    - Access Method: The host system reads this data using standard PCIe configuration read cycles. It does not require loading a specific driver; the BIOS or OS can read it immediately upon enumeration.
    - Structure: The data is organized in a specific format defined by the PCIe spec (based on the older PCI VPD standard), often using "Keywords" (e.g., `PN` for Part Number, `SN` for Serial Number, `RV` for Checksum).

3. Why is VPD Important?

    VPD is critical for several operational reasons:

    - Inventory Management: System administrators can query all devices in a server remotely to see exactly what hardware is installed (Part Numbers and Serial Numbers) without opening the chassis.
    - Driver Matching: Operating systems use VPD data to identify the exact hardware revision and load the correct drivers.
    - Thermal & Power Management: In data centers, the Baseboard Management Controller (BMC) reads VPD to understand the power draw and thermal characteristics of each NVMe drive to prevent overheating or power overload.
    - Warranty & Support: When a drive fails, support teams use the Serial Number from VPD to verify warranty status instantly.
    - Asset Tracking: Enterprises can write custom "Asset Tags" into the writable section of VPD to track which server a drive belongs to.


4. How to View VPD Data

    You can view VPD data using standard operating system tools:

    In Linux:
    The lspci command is the most common tool.

    ```
    # Show VPD for all PCIe devices
    lspci -vvv | grep -i vpd

    # Or specifically for an NVMe controller (find the bus ID first)
    lspci -xxx -s <bus_id> | grep -A 20 "VPD"
    ```
    Note: Some modern NVMe drives also expose similar identity data via the NVMe "Identify Controller" command, but VPD is the underlying PCIe standard.

    In Windows:
    You can use PowerShell or Device Manager details, but specialized tools like `lspci` ports for Windows or vendor-specific diagnostic tools are often clearer.

5. VPD vs. NVMe Identify Command

    It is important to distinguish between the two:

    - PCIe VPD: A generic standard for all PCIe devices (GPUs, NICs, SSDs). **It is read via the PCIe bus configuration space**.
    - NVMe Identify Command: A specific command within the NVMe protocol that returns much more detailed information about the SSD's capabilities (namespaces, endurance, specific features).
    - Overlap: Often, **the Serial Number and Model Number found in PCIe VPD are the same as those returned by the NVMe Identify command, but they are accessed through different layers of the stack**.

