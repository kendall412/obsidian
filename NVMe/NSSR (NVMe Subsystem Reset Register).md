
The NVMe Subsystem Reset Register (NSSR) is a critical hardware register introduced in the NVMe 1.4 specification (and expanded in later versions) to manage resets in complex, multi-controller environments typical of datacenters.

Here is a detailed breakdown of what it is, where it lives, and how it works.

1. What is the NSSR?

- Full Name: NVMe Subsystem Reset Register.
- Purpose: It allows the host software (driver/OS) to trigger a reset of the entire NVMe Subsystem, rather than just a single Controller.
- Why it matters: In modern datacenter SSDs, a single physical drive (the "Subsystem") often contains multiple independent controllers (NVM Subsystems) that share resources like power, thermal sensors, or NAND flash channels. If one controller fails catastrophically, resetting just that one might not be enough to clear the shared resource lockup. The NSSR allows the host to reset all controllers in the subsystem simultaneously to ensure a clean state.
2. Location and Structure

- Offset: `0x000C` (12 bytes from the start of the Controller Memory Space).
- Access: Write-only (Reading it typically returns 00h).
Size: 32-bit (4 bytes).


|bit|Name|Description|
|---|---|---|
|31:1|Reserved|Must be written as `0`|
|0|NSSRE|NVMe Subsystem Reset Enabled<br> `0`: No effect<br> `1`: triggers a subsystem reset|

3. How It Works (The Sequence)

    When the host driver writes a `1` to the NSSRE bit:

    1. Trigger: The NVMe controller detects the write to offset 0x000C.
    2. Propagation: The controller signals all other controllers within the same NVM Subsystem that a reset is requested.
    3. Action:
        - All controllers in the subsystem halt current operations.
        - Internal state machines are reset.
        - Volatile memory (caches, queues) is cleared.
        - The controllers behave as if they just underwent a power-on reset.
    4. Completion: The controllers re-initialize. The host must then re-run the initialization sequence (Identify, Set Features, Create Queues) for all controllers in the subsystem.

4. When is NSSR Used?

    The NSSR is primarily used in Enterprise/Datacenter scenarios:

    - **Fault Tolerance Recovery**: If a controller hits a critical error (triggering the [[FTP (Fault Tolerance Policy)]] we discussed earlier) and becomes unresponsive, the host may use NSSR to reset the whole subsystem to recover.
    - **Firmware Updates**: When updating firmware on a drive with multiple controllers, the host might use NSSR to reset all controllers simultaneously after the update to ensure they all start running the new code together.
    - **Shared Resource Deadlock**: If controllers share a resource (like a specific NAND bank or power rail) and get into a deadlock state where one controller blocks the others, a single-controller reset won't fix it. NSSR breaks the deadlock by resetting everyone.
    - **Kexec / Fast Reboot**: In Linux, kexec (rebooting into a new kernel without a full hardware reset) might use NSSR to cleanly reset the NVMe subsystem before handing control to the new kernel.

5. Difference: Controller Reset (CC.EN) vs. Subsystem Reset (NSSR)

|Feature|Controller Reset(`CC.EN`)|Subsystem Reset(`NSSR`)|
|---|---|---|
|Scope|Resets only one specific controller|Reset all controllers|
|Trigger|Clearing the`CC.EN` bit to 0|Writing `1` to `NSSR.NSSRE`.|
|Use Case|Standard driver init, single controller hang|Multi-controller deadlock, subsystem-wide faults, coordinated FW updates|
|Impact|Other controllers on the drive keep running|Everything on the drive stops and restarts|

6. Important Considerations for Drivers

    - Coordination: If a system has multiple CPU cores or sockets accessing different controllers on the same SSD, the driver must coordinate. If Core A triggers an NSSR, Core B (which might be talking to a different controller on the same drive) will suddenly see its controller disappear and reset. The driver needs a locking mechanism to prevent accidental subsystem resets during normal operation.
    - Data Loss: Like any reset, volatile write cache data that hasn't been flushed to NAND will be lost. The host should ensure all dirty data is flushed before writing to NSSR.
    - Support Check: Not all NVMe drives support this. **The host must check the Controller Capabilities (CAP) register or the Identify Controller data to see if the "Subsystem Reset Supported" bit is set before attempting to use NSSR**.

## Summary
The NSSR is the "Big Red Button" for an NVMe SSD subsystem. It is the ultimate recovery tool for datacenter drives with complex internal architectures, ensuring that when things go wrong at a subsystem level, the host can force a clean, coordinated restart of all internal components.


# Troubleshooting
## segmentation fault
#segmentation_fault
If an nvme subsystem-reset is causing a **segmentation fault**, the crash is almost always triggered by user-space software (e.g., nvme-cli or spdk_tgt) attempting to access memory addresses or kernel structs for an NVMe controller that has unexpectedly disappeared or changed states

#### Scenario 1: Using nvme-cli (Linux Terminal)<br>
If you are running `nvme subsystem-reset /dev/nvme0` followed by or during a `nvme list` command and your terminal crashes, you are likely hitting a known race condition.
- The Problem: The user-space utility crashes because it fails to gracefully handle when kernel structures are removed during the reset, resulting in a NULL pointer dereference.
- The Fix: Ensure you are running the latest version of `nvme-cli` and `libnvme`. The development teams have pushed commits that check for invalid or NULL states rather than crashing

#### Scenario 2: Using SPDK (Storage Performance Development Kit)<br>
If you are running an SPDK-based target (spdk_tgt) and triggering a subsystem reset (via NSSR or hot remove events), the process may segfault if I/O is still inflight or if detached controllers are not properly tracked.
- The Problem: In-flight I/O or pending reconnect loops collide with the NVMe subsystem reset, causing the SPDK thread to crash.
- The Fix: Pause or drain I/O explicitly from your NVMe block devices (bdev) before attempting an NSSR (NVMe Subsystem Reset) via RPC. Ensure your SPDK version is up to date, as newer releases include patches designed to handle NVMf target hot-removals more gracefully

#### Scenario 3: General System/Drive Issues<br>
If basic NVMe operations or resets cause your OS shell to freeze with segfaults, the hardware itself may be misbehaving.
- The Problem: A faulty drive, corrupt firmware, or corrupted segment mappings can trigger repeated reset attempts.
- The Fix: Check your kernel ring buffer (dmesg -T | grep -i nvme) to see if the drive is continuously entering an assert loop. Run SMART testing via smartctl -a /dev/nvme0n1 to check for media integrity or reallocated sectors.

