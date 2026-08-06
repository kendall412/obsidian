# PCIe-VDM (PCIe Vendor Defined Message)

It is a specific type of packet defined in the PCI Express (PCIe) Base Specification (not the NVMe spec itself) that allows hardware vendors (like Intel, Samsung, or Broadcom) to send custom, proprietary commands between devices or between the host and a device, bypassing standard protocol rules.

Since NVMe drives run on top of the PCIe physical layer, they can utilize PCIe-VDMs for functions that the standard NVMe command set does not support.

1. What is it technically?

    In the PCIe protocol, data is moved in packets called Transaction Layer Packets (TLPs). Most TLPs are standard (e.g., "Memory Read," "Memory Write," "Message").

    A VDM is a special type of Message TLP:

    - Standard Messages: Defined by the PCIe spec (e.g., "Power Management Event," "Error Signal"). All devices must understand these.
    - Vendor Defined Messages: The content of the message is not defined by the PCIe spec. Instead, the Vendor ID (a unique 16-bit number assigned to every manufacturer) identifies who created the message, and the vendor defines what the data inside means.

2. Structure of a PCIe VDM

    A VDM packet contains specific headers that tell the system: "This is a custom message from Vendor X."

    - Vendor ID: Identifies the manufacturer (e.g., 0x8086 for Intel, 0x144D for Samsung).
    - VDM Type: A code defined by the vendor to specify the message category.
    - Payload: The actual custom data/command being sent.
    
    There are two main routing types:

    - VDM to Bus/Device: Sent to a specific device (e.g., Host sends a custom command to the NVMe SSD).
    - VDM Peer-to-Peer: Sent between two devices directly (e.g., One NVMe SSD talking to another NVMe SSD or a GPU without CPU involvement).

3. How is VDM used in NVMe Systems?

    While the NVMe Specification defines standard commands (Read, Write, Identify, etc.), vendors sometimes need to do things outside that standard. They use PCIe-VDMs for:

    A. Out-of-Band Management
    - Scenario: The OS is crashed, or the NVMe driver isn't loaded.
    - Usage: A Baseboard Management Controller (BMC) or a specialized management tool can send a PCIe-VDM directly to the SSD to check health, update firmware, or force a reset, bypassing the standard NVMe driver stack.

    B. Proprietary Diagnostics & Telemetry

    - Scenario: An engineer needs deep internal data (e.g., specific NAND voltage levels, internal queue depth histograms) that isn't exposed in standard NVMe Log Pages.
    - Usage: The vendor provides a custom tool that sends a VDM to the drive. The drive's firmware recognizes the Vendor ID, interprets the custom command, and returns the proprietary data in the VDM response.

    C. Peer-to-Peer (P2P) Communication

    - Scenario: In high-performance computing (HPC) or AI clusters, an NVMe SSD might need to send data directly to a GPU or another SSD without copying it through the CPU's RAM.
    - Usage: PCIe VDMs can be used to set up or manage these direct P2P links, especially in complex switching fabrics where standard memory mapping is difficult.

    D. Security & Authentication

    - Scenario: A server needs to verify the authenticity of the SSD before allowing it to boot (Supply Chain Security).
    - Usage: The BIOS/BMC can send a challenge via VDM. The SSD signs the response using a private key embedded in its hardware. Since VDMs are harder to intercept or spoof than standard memory writes, this adds a layer of security.

4. Important Distinction: VDM vs. NVMe Vendor Specific Commands

    - NVMe Vendor Specific Command: Sent as a standard NVMe Admin or I/O Command (Opcode `0xFF` or reserved opcodes) through the Submission Queue. It travels over PCIe as a standard Memory Write to the SQ.
        - Pros: Works with any standard NVMe driver.
        - Cons: Requires the OS and driver to be running.
    - PCIe VDM: Sent as a raw PCIe Message Packet.
        - Pros: Can work without an OS, driver, or even a configured NVMe namespace. Lower level access.
        - Cons: Requires custom kernel drivers or hardware (BMC) to generate the raw PCIe packets. Not portable across different vendors.

5. Summary Table    
    |Fearture|Standard NVMe Command|PCIe Vendor Defined Message (VDM)|
    |---|---|---|
    |Layer|NVMe Protocol Layer (Software)|PCIe Transaction Layer (HW/Link)|
    |Transport|Memory WIrte to Submission Queue|PCIe Message TLP|
    |Standardization|Defined by NVM Express Org|Defined by Individual Vendor|

## Conclusion
PCIe-VDM is a "backchannel" for hardware vendors. It allows them to extend the capabilities of their NVMe drives beyond the limits of the official NVMe specification by sending custom raw packets over the PCIe bus. You will typically only encounter this if you are writing low-level drivers, working with BMC firmware, or using proprietary enterprise diagnostic tools.

