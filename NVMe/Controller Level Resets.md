Based on the **NVMe Base Specification 2.2**, here are the controller-level resets and the corresponding `nvme-cli` commands to mimic or trigger them.

### 1. Controller Level Reset (CC.EN Toggle)
This is the standard reset defined in the spec where the host disables the controller (`CC.EN = 0`) and then re-enables it (`CC.EN = 1`).

*   **Spec Behavior:** The controller transitions to the "Disabled" state, deletes all I/O and Admin queues, and clears internal state. When re-enabled, it performs a full initialization.
*   **How to Mimic in `nvme-cli`:**
    There is no single "reset" command that does exactly this toggle in user space without potentially disconnecting the drive. However, the **`nvme reset`** command is designed to perform a controller level reset.

    ```bash
    sudo nvme reset /dev/nvme0
    ```
    *   **What it does:** This command attempts to toggle the `CC.EN` bit.
    *   **Warning:** This will disconnect all active namespaces and I/O queues. If you are running this on the boot drive or a drive with an active filesystem, the system may hang or the filesystem may need remounting.

### 2. NVMe Subsystem Reset (NSSR)
Defined for subsystems containing multiple controllers, but applicable to single controllers that support the **NVMe Subsystem Reset Register (NSSR)**.

*   **Spec Behavior:** Writes to the `NSSR.SSN` (Subsystem Shutdown Notification) or triggers the reset bit to reset the entire subsystem.
*   **How to Mimic in `nvme-cli`:**
    Use the `subsys-reset` command.

    ```bash
    sudo nvme subsystem-reset /dev/nvme0
    ```
    *   **Note:** Not all standalone NVMe drives support this register. It is more common in enterprise U.2/U.3 drives or NVMe fabrics. If the controller doesn't support it, the command will fail with an error indicating the feature is not supported.

### 3. [[Power State]] Reset (Power Cycle)
While technically a power management event, the spec treats a transition to a non-operational power state (and back) as a reset event for volatile state.

(see [[how to tie nvme drive to bus number]])

*   **Spec Behavior:** The controller enters a low-power state (often losing volatile configuration) and re-initializes upon returning to an operational state.
*   **How to Mimic in `nvme-cli`:**
    You can force a power state transition using the `set-feature` command for **Power State (Feature ID 0x02)**.

    **Step A: Check supported power states**
    ```bash
    sudo nvme get-feature /dev/nvme0 -f 0x02 -H
    # Or view the identify controller data for power state descriptors
    sudo nvme id-ctrl /dev/nvme0 | grep -A 20 "ps"
    ```

    **Step B: Set a non-operational power state (e.g., State 2 or 3, usually lowest power)**
    *Replace `2` with a valid non-operational state ID from your drive's support list.*
    ```bash
    sudo nvme set-feature /dev/nvme0 -f 0x02 -v 2
    ```
    *   **Result:** The drive may disappear from the bus or become unresponsive until the host resets the link or the drive wakes up.
    *   **Alternative (Physical Power Cycle):** The most reliable way to mimic a full power-on reset (POR) is physically removing power or using a system-level reboot:
        ```bash
        sudo reboot
        ```

### 4. PCIe Hot Reset (Transport Level)
The NVMe spec mandates that the controller must handle a PCIe Hot Reset as a reset event.

*   **Spec Behavior:** The PCIe link goes down and comes back up. The NVMe controller must re-initialize as if it were powered on.
*   **How to Mimic in `nvme-cli`:**
    `nvme-cli` itself does not have a direct "PCIe Hot Reset" command because this is a PCIe bus operation, not an NVMe register operation. However, you can achieve this via the **sysfs** interface (Linux kernel):

    **Method A: Remove and Rescan the PCIe device**
    ```bash
    # Find the PCI address (e.g., 0000:01:00.0)
    lspci | grep -i nvme

    # Remove the device (triggers link down/reset)
    echo 1 | sudo tee /sys/bus/pci/devices/0000:01:00.0/remove

    # Rescan the bus to bring it back (triggers re-enumeration and controller reset)
    echo 1 | sudo tee /sys/bus/pci/rescan
    ```
    *   **Warning:** This is extremely disruptive. If `/dev/nvme0` is your root filesystem, the system will crash.

### Summary Table

| Reset Type | Spec Mechanism | `nvme-cli` Command | Risk Level |
| :--- | :--- | :--- | :--- |
| **Controller Reset** | Toggle `CC.EN` | `sudo nvme reset /dev/nvmeX` | High (Disconnects queues) |
| **Subsystem Reset** | Write `NSSR` | `sudo nvme subsystem-reset /dev/nvmeX` | High (Support dependent) |
| **Power State Reset** | Change Power State | `sudo nvme set-feature -f 0x02 -v <state>` | Medium/High (May require reboot) |
| **PCIe Hot Reset** | PCIe Link Reset | `echo 1 > /sys/.../remove` + `rescan` | Critical (System crash if boot drive) |

### Important Safety Warning
*   **Data Integrity:** Running these commands on a drive with a mounted filesystem or active I/O will almost certainly cause **I/O errors, filesystem corruption, or system crashes**.
*   **Test Environment:** Always perform these resets on a non-boot drive in a test environment.
*   **Namespace Impact:** Controller-level resets delete all I/O Submission and Completion queues. The OS will lose communication with the namespaces until the driver re-initializes the controller.


## Theory

#### NVMe Base Spec v2.2 3.7.2 Controller Level Reset pg 116

The following methods initiate a Controller Level Reset:
* NVM Subsystem Reset
* Controller Reset (_i.e._, host clears the **`CC.EN` bit from `1` to `0`**)
* Transport specific reset types (refer to the applicable NVMe Transport binding specification), if any.

A Controller Level Reset consists of the following actions:
* The controller stops processing any outstanding Admin or I/O commands,
* All I/O Submission Queues are deleted
* ALl I/O Completion Queues are deleted; and
* The controller is brought to an idel state. When this is complete, the **`CSTS.RDY` bit is cleared to `0`**; and
* All NVMe controller properties defined in either section 3.1.4 or the applicable NVMe Transport binding specification and all internal controlelr state are reset, with the following exceptions:
    * for memory-based controller:
        * the following are not reset as part of a Controller Level Reset caused by a Controller Reset:
            - Admin Queue properties (_i.e._ AQA, ASQ, and ACQ)
            - Persistent memory Region properties (_i.e._ PMRCAP, PMRCTL, PMRSTS, PMREBS, PMRSWTP, PMRMSCU, and PMRMSCL); and
            - The Controller Memory Buffer Memory Space Control property (CMBMSC) (refer to the NVM Express NVMe over PCIe Transport Specification);
        * the following are not reset as part of a Controller Level Reset caused by a Function Level Reset:
            - the Controller Memory Buffer Memory Space Control property (CMBMSC);
    * for message-based controllers:
        * there are no exceptions

In all Controller Level Reset cases except a Controller Reset, the controller properties defined by the transport (_e.g._ the PCIe registers defined by the PCIe Base Specification) are reset as defined by the applicable NVMe Transport binding specification (_e.g._ the NVM Express NVMe over PCIe Transport Specification).

In all Controller Level Reset cases, if the media is not usable and an NVM Subsystem Shutdown that includes the controller is neither reported as in progress nor reported as complete (_i.e._ the `CSTS.ST` bit is cleared to `0` or the `CSTS.SHST` field is cleared to `00b`), then the controller is permitted to initialize the media for use upon completion of the Controller Level Reset.

To continue after a Controller Level Reset, the host should:
* update transport specific state and controller property state as appropriate;
* set the **CC.EN bit to `1`**;
* wait for the **CSTS.RDY bit to be set to `1`**;
* configure the controller using Admin commands as needed;
* create I/O Completion Queues and I/O Submission Queues as needed; and
* proceed with normal I/O operations.

Note that all Controller Level Reset cases except a Controller Reset result in the controller immediately losing communication with the host. In all these cases, the controller is unable to indicate any aborts or update any completion queue entries.

If the host is no longer able to communicate with the controller before that host receives either:
* completions for all outstanding commands submitted to that controller (refer to section 3.4.5); or
* a `CSTS.RDY` bit value cleared to `0`,

then it is strongly recommended that the host take the steps described in section 9.6 to avoid possible data corruption caused by interaction between outstanding commands and subsequent commands submitted by that host to another controller.
