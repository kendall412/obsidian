# Persistent 

means that a specific setting, configuration, or piece of data is saved to non-volatile memory (NVM) on the drive itself and survives a power cycle or reset.

When a feature is "persistent," you set it once, and the drive remembers it forever (until you explicitly change it again), even if you unplug the power or reboot the server.

This is in direct contrast to Volatile settings, which are stored only in the controller's temporary RAM and are lost immediately when power is cut or the controller resets.

1. The Two States: Volatile vs. Persistent

    |Fearture State|Storage Location|Behavior On Power Loss/Reset|Typical Use Case|
    |---|---|---|---|
    |Volatile|Controller DRAM/SRAM|Lost. Reverts to factory default or previous saved state|Themporary tuning, performance testing, dynamic power management|
    |Persistent|NAND Flash / Non-Volatile Memory|Retained. The drive reloads this setting automatically on boot|Security keys, namespace configuratins, cirtical erro policies, WWN/NGUID|

2. How "Persistent" Works in Commands

    The most common place you encounter this concept is in the Set Features command.

    When a host sends a [[Set Features Command]] to change a setting (like Write Cache, Interrupt Coalescing, or Fault Tolerance), there is a specific bit in the command called the **Save (SV) bit (Bit 31 of CDW10)**.

    - If SV = 0 (Volatile):
        - The controller applies the change immediately.
        The change is NOT saved to flash.
        - Result: If the drive loses power, this setting is forgotten. On the next boot, the driver must re-send the command to restore the setting.
        - Usage: Most performance tuning (e.g., interrupt coalescing) is volatile because the OS driver handles re-applying it every boot anyway.
    - If SV = 1 (Persistent):
        - The controller applies the change immediately.
        - The controller writes the new value to its internal non-volatile storage.
        - Result: If the drive loses power, the setting is remembered. On the next boot, the controller automatically loads this value without needing the host to send the command again.
        - Usage: Critical configurations like Security Keys, Namespace creation/deletion, or specific Fault Tolerance Policies.

    Important Note: Not all features support the "Save" (Persistent) option. If you try to set SV=1 for a feature that doesn't support it, the drive will return an error (`Invalid Field` or `Save Field Not Supported`). You must check the "Saveable" column in the NVMe Spec's Features table.

3. Common Persistent Elements in NVMe

    A. Namespaces

    When you create a Namespace (using the Namespace Management command), the configuration (size, NGUID, EUI64, protection information) is persistent.

    - If you unplug the drive and plug it into a different server, the namespace still exists with the same data and ID.
    - Deleting a namespace is also a persistent action; the space is freed permanently.

    B. Security Settings

    - Password Locks: If you set a password on an NVMe drive (Security Send/Receive), this is persistent. The drive will remain locked even after a power cycle until the correct password is provided.
    - Crypto Erase: Marking a drive for a cryptographic erase is a persistent command. The drive remembers to wipe its keys on the next reset.

    C. Firmware

    - When you update the firmware (Firmware Commit), you can choose to **activate it immediately (volatile activation)** or **commit it to the "Slot" to be activated on the next reboot (persistent activation)**.

    D. [[FTP (Fault Tolerance Policy)]]

    - As discussed earlier, **while many drivers set FTP volatily at boot, enterprise drives often allow you to save the FTP setting persistently**. This ensures that even if the OS fails to load or the driver crashes during boot, the drive defaults to a safe error-handling mode defined in its own memory.

4. Why "Persistent" Matters in Datacenters

    1. Boot Independence: If a server's OS is corrupted or the driver fails to load, persistent settings (like security locks or namespace maps) ensure the data remains protected and structured according to the last known good configuration.
    2. Drive Swapping: In a hot-swap scenario, if you move a drive from Server A to Server B, Server B can immediately recognize the namespaces and security state because that data is persistent on the drive, not stored on Server A's motherboard.
    3. Compliance: For security standards (like FIPS), certain configurations (e.g., "FIPS Mode Enabled") must be persistent to prove that the drive cannot accidentally revert to an insecure state after a power glitch.

## Summary
In NVMe, **Persistent = "Saved to Flash, Survives Power Loss."**
It is the mechanism that allows the SSD to act as an intelligent, stateful device that remembers its configuration, security state, and namespace layout independently of the host computer it is plugged into.
