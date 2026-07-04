
> In PCIe **out-of-band (OOB) management**, **management data** is **information about the device itself**, rather than the user's application data.

For an NVMe SSD:

- **User data** = The contents of files, databases, virtual machine images, etc.
- **Management data** = Information about the SSD's health, configuration, status, and maintenance.

## Two Types of Data

Imagine an NVMe SSD storing a video file.

### User Data

```
movie.mp4

Block 1000
Block 1001
Block 1002
```

This is what the host reads and writes using NVMe Read/Write commands.

### Management Data

Information about the SSD itself:

```
Temperature = 42°C

Power-On Hours = 12,345

Firmware Version = 2.1.7

Media Errors = 0

Available Spare = 98%
```

This is management data.

## In-Band vs Out-of-Band

### In-Band

```
CPU
 |
PCIe
 |
NVMe SSD
```

Uses:

- Read
- Write
- Identify
- Get Log Page

Everything goes through PCIe TLPs.

### Out-of-Band

```
BMC
 |
SMBus
 |
NVMe-MI
 |
NVMe SSD
```

Uses:

- Temperature
- Health
- Inventory
- Diagnostics

No PCIe packets are involved.

### Examples of Management Data

#### 1. Temperature

BMC asks:

```
Current SSD temperature?
```

SSD replies:

```
43°C
```

#### 2. Firmware Version

Request:

```
Firmware Revision?
```

Response:

```
FW 3.2.1
```

#### 3. Model Information

Response:

```
Model

XYZ Enterprise SSD

Capacity

3.84 TB

Serial Number

ABC123456789
```

#### 4. Health

Example:

```
Available Spare

95%
```


#### 5. Power State

Example:

```
Current NVMe Power State

PS3
```

#### 6. Critical Warning

Example:

```
Critical Warning

Temperature Warning

No
```

#### 7. Lifetime

Example:

```
Power-On Hours

18,240
```

#### 8. Error Information

Example:

```
Media Errors

2

Unsafe Shutdowns

4
```

#### 9. Inventory

Example:

```
PCI Vendor

Samsung

Model

PM9A3

Firmware

GDC5902Q
```

#### 10. Device Status

Example:

```
Controller Ready

Yes

Temperature Normal

Yes

Background Scan

Running
```

## Real Enterprise Server

Suppose a server has:

```
8 NVMe SSDs
```

Every minute, the BMC polls:

```
SSD0

Temperature

41°C

Life

98%
```

```
SSD1

Temperature

39°C

Life

96%
```

No file data is transferred. Only management information.

### Example: User Data

Application requests:

```
Read File
```

Flow:

```
Application

↓

OS

↓

NVMe Driver

↓

PCIe

↓

SSD

↓

DMA

↓

File Contents
```

### Example: Management Data

Administrator asks:

```
Show SSD Temperature
```

Flow:

```
BMC

↓

SMBus

↓

NVMe-MI

↓

SSD

↓

Temperature
```

The file contents are never accessed.

## Why Enterprise Servers Need This

Suppose:

```
Operating System

↓

Crash
```

The host cannot issue:

```
Get Log Page
```

But the BMC still wants to know:

```
Is SSD alive?

Temperature?

Overheating?

Firmware Version?
```

Out-of-band management provides this information independently of the operating system.

## Typical Management Information

|Category|Example|
|---|---|
|Health|Temperature, spare capacity, endurance|
|Inventory|Model, serial number, firmware|
|Status|Current power state, controller state|
|Errors|Media errors, error logs|
|Statistics|Power-on hours, unsafe shutdown count|
|Diagnostics|Internal health information|
|Firmware|Version and update status|

## What Is NOT Management Data?

The following are **user data**, not management data:

```
Photos

Videos

Databases

Virtual Machines

Documents

Operating System Files
```

These always travel through the PCIe data path.

## Visualization

```
                     SSD

             +------------------+
             |                  |
             | User Data        |
             |                  |
             +------------------+
                     ^
                     |
                  PCIe
                     |
                    CPU


             +------------------+
             | Management Data  |
             |                  |
             +------------------+
                     ^
                     |
              NVMe-MI / SMBus
                     |
                    BMC
```

Notice there are two completely different communication paths.

## In-Band Management vs Out-of-Band Management

One subtle but important point is that **the same information can often be accessed through either path**.

For example, SSD temperature can be read:

### In-Band

```
CPU

↓

PCIe

↓

NVMe Admin Command

↓

Temperature
```

### Out-of-Band

```
BMC

↓

SMBus

↓

NVMe-MI

↓

Temperature
```

The information is the same. The communication path is different.

# Summary

In PCIe out-of-band management, **management data** is information **about the device**, not the user's stored data.

Typical management data includes:

- Device identity (model, serial number, firmware revision)
- Health (temperature, endurance, available spare)
- Status (controller state, power state)
- Statistics (power-on hours, unsafe shutdowns)
- Error information (media errors, diagnostics)
- Firmware management information

For an NVMe SSD:

- **User data** travels over the **PCIe data path** using NVMe I/O commands.
- **Management data** travels over the **out-of-band management path** (typically **SMBus + NVMe-MI**) so that a BMC or management controller can monitor and administer the SSD even when the normal storage software stack is unavailable.