# Asynchronous Event

> An Asynchronous Event in NVMe is a notification mechanism that allows an SSD to proactively alert the host system (OS, driver, or BMC) when a significant condition occurs, without the host needing to constantly check (poll) the drive's status. It is the storage equivalent of a fire alarm: instead of a security guard checking every room every minute to see if there is a fire (polling), the alarm rings immediately when smoke is detected (asynchronous event).

1. Why It Exists (The Problem with Polling)

    In older storage protocols or naive implementations, the host had to repeatedly ask the drive:

    - "Are you hot?"
    - "Is your space full?"
    - "Did an error occur?"
    
    Problems with Polling:

    - Wasted CPU: The host CPU spends cycles running checks that usually return "Everything is fine."
    - Latency: If an error happens between checks, the host might not know for seconds or minutes, which is too late for critical data protection.
    - PCIe Bandwidth: Constant polling commands consume bandwidth on the PCIe bus.

    The Asynchronous Solution:

    > The host submits a "Listen" command once and goes to sleep. The drive holds that command open. When an event happens, the drive completes that command instantly to wake up the host.

2. How It Works (The "Submit and Wait" Cycle)

    The mechanism relies on the **Admin Submission Queue** and **Admin Completion Queue**.

    1. Host Submits Request: The host sends an Asynchronous Event Request (AER) command (Opcode `0x0C`) to the drive. This command has no data buffer; it simply says, "Notify me when something happens."
    2. Drive Holds Command: The drive receives the command but does not complete it immediately. It places the command in a pending state. (Best practice is for the host to keep 2–3 AER commands pending at all times to ensure no events are missed).
    3. Event Occurs: Something happens inside the SSD (e.g., temperature hits 80°C, a NAND block fails, or power loss protection activates).
    4. Drive Notifies: The drive immediately completes one of the pending AER commands.
        - It places a Completion Entry in the Admin Completion Queue.
        - This entry contains a specific Event Type and Event Info code describing what happened.
        - It triggers an interrupt (MSI-X) to the host CPU.
    5. Host Reacts:
        - The host driver sees the interrupt.
        - It reads the Completion Queue to see what happened.
        - It typically issues a `Get Log Pag` command to fetch detailed data (e.g., reading the Smart Health Log to see the exact temperature).
    6. Re-Arm: The host immediately submits a new AER command to replace the one that was just completed, restarting the cycle.

3. Types of Asynchronous Events

    The NVMe specification categorizes events into three main groups. The completion message tells the host which group triggered the alert.

    A. Error Events (Critical)
    - Endurance Exceeded: The drive has reached its total write limit (TBW).
    - Uncorrected Error: A read/write failed and could not be fixed by ECC.
    - Volatile Memory Backup Failed: The capacitors (PLP) failed to save data during a power loss.
    - Write Fault: The drive can no longer accept writes (entering Read-Only mode).

    B. Smart / Health Events (Warning)
    - Temperature Threshold: Temperature crossed a configured Warning or Critical limit.
    - Available Spare Space: The reserved NAND pool dropped below a threshold (_e.g._, < 10%).
    - Device Reliability: The "Percentage Used" indicator reached a configured limit (_e.g._, 90% worn out).
    - Read Only: The drive has switched to read-only mode.
    
    C. Notice Events (Informational)
    - Namespace Attribute Changed: A namespace was created, deleted, or resized.
    - Firmware Activation Started: A new firmware image is ready to be activated (often requires a reset).
    - Reservation Preempted: In multi-host systems, another host took over a locked resource.
    - Power State Change: The drive entered a specific power saving mode.

4. Why It Is Critical for OCP & Data Centers <!-- tags: #ocp -->

    In the context of OCP (Open Compute Project) and hyperscale clouds, Asynchronous Events are mandatory for automation:

    1. Predictive Maintenance: When a drive sends an "Uncorrected Error" AER, the cloud software can instantly migrate data off that drive before it fails completely, preventing data loss for the customer.
    2. Thermal Management: If a drive sends a "Critical Temperature" AER, the BMC can instantly ramp up fan speeds or throttle the CPU to prevent hardware damage.
    3. Resource Efficiency: A single BMC can manage thousands of drives without wasting CPU cycles polling them every second. It only wakes up when a drive actually needs attention.
    4. SLA Compliance: Cloud providers guarantee uptime. AER allows them to react to issues in milliseconds rather than minutes, maintaining their Service Level Agreements.

5. Practical Example (Linux `nvme-cli`)

    You can see AER in action using Linux tools. When a drive triggers an event, the kernel log (`dmesg`) will often show it:

    ```
    # Example kernel message when an AER occurs
    
    nvme nvme0: Async Event Request completed (Result: 0x00000000, CID: 0x12)
    
    # The driver then automatically fetches the log page
    nvme nvme0: Smart/Health status changed, reading log...
    ```
    
In short, Asynchronous Events are the nervous system of an NVMe drive, allowing it to communicate critical health and status changes to the host instantly and efficiently.






 






