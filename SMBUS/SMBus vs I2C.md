
> **SMBus (System Management Bus)** and **I²C (Inter-Integrated Circuit)** are closely related protocols, but they are **not identical**. **SMBus is a standardized subset of I²C designed specifically for computer system management.** Most SMBus devices understand I²C signaling, but not every I²C device fully complies with SMBus timing and protocol requirements.

## High-Level Comparison

|I²C|SMBus|
|---|---|
|General-purpose communication bus|System management bus|
|Developed by NXP Semiconductors|Developed by System Management Bus Forum (based on I²C)|
|Used in many embedded systems|Used mainly in PCs and servers|
|Flexible specification|More strictly defined|
|Sensors, EEPROMs, displays, ICs|Batteries, temperature sensors, PCIe/NVMe management, BMCs|

## Physical Layer

Both buses use exactly two signals:

```
Clock
Data
```

I²C:

```
SCL
SDA
```

SMBus:

```
SMCLK
SMDAT
```

Electrically they look very similar.

```
      +3.3V
        |
    Pull-up
        |
SCL -----+-------------------

      +3.3V
        |
    Pull-up
        |
SDA -----+-------------------
```

SMBus uses the same open-drain concept.

## Basic Communication

Both protocols use:

```
START
Address
Read/Write
Data
ACK
STOP
```

Example:

```
START

Address

ACK

Register

ACK

Data

ACK

STOP
```

The communication sequence is almost identical.

## Major Differences

### 1. Purpose

#### I²C

General communication.

Example:

```
Microcontroller
    |
    +------ EEPROM
    |
    +------ LCD
    |
    +------ Accelerometer
```

Used almost everywhere.

#### SMBus

Computer management.

Example:

```
BMC
 |
SMBus
 |
NVMe SSD
 |
Temperature Sensor
 |
Power Supply
```

Purpose:

- Health monitoring
- Inventory
- Battery management
- Firmware management

### 2. Timing Requirements

SMBus defines stricter timing.

Example:

SMBus specifies:

- Minimum clock frequency
- Maximum clock frequency
- Bus timeout

I²C allows much more flexibility.

### 3. Timeout Feature

This is one of the biggest differences.

#### I²C

If the clock stops:

```
SCL held LOW
```

the bus can remain stuck forever.

#### SMBus

If the clock remains low longer than the timeout period (approximately **35 ms** in the SMBus specification), devices assume the transaction has failed and recover.

Example:

```
Master crashes

Clock stays LOW

35 ms

Bus resets internally
```

This prevents a permanently hung management bus.

### 4. Voltage Levels

SMBus specifies tighter electrical requirements. I²C allows wider voltage ranges.

### 5. Packet Error Checking (PEC)

SMBus optionally supports:

```
PEC
```

(Packet Error Code)

This is an additional CRC byte.

Example:

```
AddressCommandDataPEC
```

The receiver verifies the PEC before accepting the transaction. **Standard I²C does not define PEC.**

### 6. Defined Commands

I²C simply transfers bytes. SMBus defines standardized commands.

Example:

```
Read Byte

Write Byte

Read Word

Write Word

Block Read

Block Write
```

These standard command formats improve interoperability.

### Speed

#### I²C

Supports many speeds.

|Mode|Speed|
|---|---|
|Standard|100 kHz|
|Fast|400 kHz|
|Fast+|1 MHz|
|High Speed|3.4 MHz|

#### SMBus

Typically:

|Mode|Speed|
|---|---|
|Standard|100 kHz|
|Fast|400 kHz|

SMBus focuses on reliable management rather than maximum speed.

## SMBus in PCIe Systems

Typical server:

```
               PCIe

CPU ---------------- SSD
        |
        |
       BMC
        |
      SMBus
        |
      SSD
```

The CPU performs:

```
PCIe Read
PCIe Write
DMA
```

The BMC performs:

```
Temperature
SMART
Inventory
Firmware Status
```

through SMBus.

### Example: Reading Temperature

Using SMBus:

```
START

Address

ACK

Temperature Register

ACK

Repeated START

Address

ACK

Temperature Data

ACK

STOP
```

Very similar to I²C.

## NVMe-MI

Many enterprise NVMe SSDs support **NVMe Management Interface (NVMe-MI)**. NVMe-MI often runs over SMBus.

Example:

```
BMC
 |
SMBus
 |
NVMe SSD
```

Commands include:

- Read temperature
- Read SMART information
- Read firmware version
- Read inventory

without using PCIe.

## Compatibility

Many controllers support both protocols.

```
I²C Controller

Compatible with

SMBus Devices
```

However:

- An SMBus master can often communicate with many I²C devices.
- An arbitrary I²C device is **not guaranteed** to meet SMBus requirements (especially timeout behavior and electrical specifications).

### Example

Laptop motherboard:

```
CPU
PCIe
NVMe SSD
```

Data transfer:

```
PCIe
```

Management:

```
Embedded Controller

SMBus

Battery
Temperature Sensors
Power IC
```

Server:

```
BMC

SMBus

NVMe SSD
Retimer
Power Supplies
Fans
```

# Summary

|Feature|I²C|SMBus|
|---|---|---|
|Origin|General embedded bus|Computer management bus|
|Signals|SCL, SDA|SMCLK, SMDAT|
|Physical layer|Very similar|Based on I²C|
|Timeout|No required timeout|Required timeout (~35 ms)|
|Packet Error Code|Not defined|Optional PEC|
|Timing|Flexible|Strictly specified|
|Primary Use|General peripherals|System monitoring and management|
|PCIe/NVMe Use|Rare directly|Common for BMC and NVMe-MI|

## In PCIe and NVMe systems

- **PCIe** carries high-speed data such as NVMe read/write commands.
- **SMBus** provides a low-speed **sideband management** path.
- **NVMe-MI** commonly uses SMBus so a BMC can monitor and manage an NVMe SSD independently of the PCIe data path.
