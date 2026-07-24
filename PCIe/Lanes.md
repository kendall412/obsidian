> PCIe lanes are the individual data pathways that allow components like graphics cards and SSDs to communicate with a computer's motherboard. They work like a highway, with more lanes providing more bandwidth and faster data transfer rates; a single lane is designated as "x1," while a group of 16 is "x16". Each of the lanes uses two wires for sending and two for receiving information simultaneously, hence greater potential for faster exchange. The number of lanes available is determined by the processor and chipset, and they are used for high-speed connections to the CPU or other components.  

# What is a PCIe Lane?

A lane consists of:

```
TX+  TX-
RX+  RX-
```

One lane can transmit and receive simultaneously (full duplex).

Examples:

|Link Width|Number of Lanes|
|---|---|
|x1|1 lane|
|x2|2 lanes|
|x4|4 lanes|
|x8|8 lanes|
|x16|16 lanes|

For example, most NVMe SSDs use:

```
PCIe x4
```

meaning four lanes.

# Why Does LTSSM Need to Determine Link Width?

Consider:

```
Motherboard Slot = x4
NVMe SSD = x4
```

Expected result:

```
Negotiated Width = x4
```

But if one lane is damaged:

```
Lane 0 = Good
Lane 1 = Good
Lane 2 = Good
Lane 3 = Failed
```

The LTSSM may train the link as:

```
x3
```

(or x2/x1 depending on the implementation).

The LTSSM automatically determines how many usable lanes exist.

# Where in LTSSM Does Link Width Get Determined?

During the early training states:

```
Detect
   |
Polling
   |
Configuration
   |
L0
```

Most width negotiation occurs in:

```
Configuration State
```

# Example: x4 NVMe SSD

Host Root Complex:

```
Lane 0
Lane 1
Lane 2
Lane 3
```

SSD:

```
Lane 0
Lane 1
Lane 2
Lane 3
```

During training, both sides exchange:

```
TS1 Ordered Sets
TS2 Ordered Sets
```

These contain information about:

- Lane number
- Link number
- Desired width
- Training status

# TS1 / TS2 Exchange

Simplified:

Host sends:

```
TS1:
Lane 0
Lane 1
Lane 2
Lane 3
```

SSD responds:

```
TS1:
Lane 0
Lane 1
Lane 2
Lane 3
```

After successful negotiation:

```
Link Width = x4
```

# Example: Width Downgrade

Suppose Lane 3 is bad.

Host sees:

```
Lane 0 OK
Lane 1 OK
Lane 2 OK
Lane 3 Missing
```

SSD sees the same.

LTSSM may decide:

```
Link Width = x2
```

or

```
Link Width = x1
```

depending on supported lane groupings.

This is called **link width degradation** or **width down-training**.

# How Width Appears Logically

A PCIe x4 link looks like:

```
Lane 0
Lane 1
Lane 2
Lane 3
```

Data is striped across lanes:

```
Packet Data

Byte0 -> Lane0
Byte1 -> Lane1
Byte2 -> Lane2
Byte3 -> Lane3
```

Result:

```
Higher Bandwidth
```

than x1.

# [[Link Training and Status State Machine (LTSSM)]] and Lane Detection

During training, the PHY performs:

```
Receiver Detection
```

on each lane.

Example:

```
Lane0 -> Receiver Found
Lane1 -> Receiver Found
Lane2 -> Receiver Found
Lane3 -> Receiver Found
```

Result:

```
Candidate Width = x4
```

# Link Width Registers

After training completes, software can read the negotiated width.

In the PCIe Capability structure:

[[LnkSta (Link Status Register)]]

Contains:

```
Negotiated Link Width
```

Example:

```
Link Width = x4
```

Linux example (bash):

```
lspci -vv
```

Output:

```
LnkSta:
Speed 16GT/s
Width x4
```


# Relationship Between Link Width and Speed

These are different.

### Link Width

Number of lanes:

```
x1
x2
x4
x8
x16
```

### Link Speed

Speed per lane:

```
Gen1 = 2.5 GT/s
Gen2 = 5.0 GT/s
Gen3 = 8.0 GT/s
Gen4 = 16 GT/s
Gen5 = 32 GT/s
Gen6 = 64 GT/s
```

Example:

```
Gen4 x4
```

means:

```
4 lanes
16 GT/s per lane
```

# NVMe Example

A PCIe Gen4 NVMe SSD:

```
M.2 SSD
```

typically trains as:

```
Gen4 x4
```

During LTSSM:

```
Detect
Polling
Configuration
Recovery
L0
```

the controller and root complex negotiate:

```
Width = x4
Speed = Gen4
```

before entering:

```
L0 (Normal Operation)
```

# Link Width in LTSSM Debugging

Firmware and validation engineers often debug:

### Expected

```
Gen4 x4
```

### Actual

```
Gen4 x2
```

or

```
Gen1 x1
```

Possible causes:

- Bad PCB trace
- Connector issue
- Signal integrity problem
- Equalization failure
- Faulty lane
- PHY configuration bug


# Summary

**Link Width** is the number of PCIe lanes that become active after LTSSM training.

Examples:

```
x1 = 1 lane
x4 = 4 lanes
x8 = 8 lanes
x16 = 16 lanes
```

During the **Configuration** portion of the LTSSM:

1. Each side detects available lanes.
2. TS1/TS2 ordered sets are exchanged.
3. Lane numbering and alignment are established.
4. A common link width is negotiated.
5. The link enters **L0** with the negotiated width.

For a typical NVMe SSD, the desired result is:

```
PCIe Gen4 x4
```

meaning **4 active lanes operating at Gen4 speed**.

---
How PCIe lanes work

Data transmission: Each lane is a pair of wires that allows for a full-duplex connection, meaning it can send and receive data at the same time. 

Bandwidth: The total number of lanes in a connection determines its overall bandwidth. A higher number of lanes (e.g., x16) allows for more data to be transferred simultaneously than a smaller number (e.g., x4). 

Slot size: The physical slot on the motherboard corresponds to the number of lanes it can support. For example, a graphics card is typically installed in a long x16 slot, while a smaller M.2 SSD uses an x4 slot. 

Hierarchy: Lane allocation is often determined by the CPU, which has a limited number of lanes directly connected to it, and the chipset, which handles other peripherals. 

  

Speed: The speed of each lane is also dependent on the PCIe generation. For example, a single lane in PCIe 4.0 is twice as fast as a single lane in PCIe 3.0. 

Why they are important

  

Performance: A device's performance is often limited by the number of lanes it can use. A graphics card in an x16 slot will have more available bandwidth than if it were placed in an x8 slot. 

  

Flexibility: You can install a device with fewer lanes than the slot supports, and the connection will simply operate at the device's lower speed. Conversely, you can install a PCIe 3.0 device in a PCIe 4.0 slot, and it will work, but at the slower PCIe 3.0 speed. 

  

Device compatibility: The number of lanes a device requires is determined by its intended use. High-performance devices like high-end GPUs and NVMe SSDs need more lanes for optimal performance compared to other peripherals.