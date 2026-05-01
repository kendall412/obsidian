
The **NVMe Admin Command Set** is the **mandatory control interface** used to initialize, configure, and manage an NVMe controller and its namespaces. It does **not move user data**—it handles everything around the data path. These commands are essential for setting up and managing the device. (see also [[Admin Command Set OPCODE (OPC)]])

### Core Purpose

- Bring the controller online
- Configure queues and features
- Discover device capabilities
- Manage firmware and namespaces

👉 Think of it as the **control plane** of NVMe.

### Where It Runs

- Executed through the **Admin Submission Queue (ASQ)**
- Completions returned via the **Admin Completion Queue (ACQ)**
- Typically only **one admin queue pair per controller**

## Command Categories

1. Controller Initialization & Setup
Create I/O Submission/Completion Queues
Delete I/O Queues

👉 Sets up the high-performance data path.

2. Device Identification
Identify
Controller details
Namespace information
Supported features and limits

3. Feature Management
Get Features
Set Features

Examples:

Power management
Number of queues
Interrupt configuration

4. Namespace Management
Create/Delete Namespace
Attach/Detach Namespace

👉 Controls how storage is logically presented.

5. Firmware Management
Firmware Image Download
Firmware Commit

👉 Enables updates without removing the device.

6. Format & Sanitize
Format NVM
Sanitize

👉 Used for:

Resetting storage
Secure erase operations


7. Log & Health Monitoring
Get Log Page

Provides:

SMART/health data
Error logs
Performance statistics


```
Admin Command Set → Configures controller
        ↓
Creates I/O Queues
        ↓
I/O Command Sets → Perform actual data reads/writes
```



Key characteristics:
Sent through a dedicated Admin Queue
Used during initialization and configuration
- **Mandatory** → Every NVMe device must support it
- **Low frequency** → Used during setup or management, not heavy I/O
- **Small data transfers** compared to I/O commands
- **Synchronous control operations** (relative to I/O workload)

Common Admin Commands:
Identify → Get device information (model, capabilities, namespaces)
Create/Delete I/O Queue → Set up queues for data operations
Get/Set Features → Configure device behavior
Firmware Download/Commit → Update firmware
Format NVM → Format storage

