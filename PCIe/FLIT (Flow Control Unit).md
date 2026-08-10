#pcie 
In **PCIe**, a **FLIT** means **Flow Control Unit**. It is a fixed-size data block used to carry PCIe traffic across the link.

## In simple terms

A **FLIT** is like a fixed-size container that PCIe uses to transport packets.
Older PCIe generations mainly send variable-sized packets such as:

- **TLPs** — Transaction Layer Packets
- **DLLPs** — Data Link Layer Packets

With newer PCIe, especially **PCIe 6.0**, these packets are packed into **fixed-size** [[FLIT (Flow Control Unit)]].

## PCIe FLIT size

In **PCIe 6.0**, a FLIT is:
#pcie_gen_6 
```text
256 bytes
```

It can contain PCIe packet data plus protection/management information such as [[CRC (Cyclic Redundancy Check)]] and [[FEC (Forward Error Correction)]] related data.

## Why PCIe uses FLITs
#flit
PCIe 6.0 introduced **PAM4 signaling** to reach **64 GT/s**. PAM4 is faster but more error-prone than older NRZ signaling.

FLIT mode helps because it works well with:

- [[FEC (Forward Error Correction)]] #fec
- Fixed-size data framing
- Better link efficiency
- Easier error detection and correction
- More predictable flow control

## FLIT vs TLP
#tlp

| Term | Meaning |
|---|---|
| **TLP** | PCIe transaction packet, such as memory read/write |
| **DLLP** | Data link packet, used for acknowledgments, credits, etc. |
| **FLIT** | Fixed-size transport container that carries TLPs/DLLPs |

So:

```text
TLP/DLLP data is packed into FLITs for transmission
```

## Simple analogy

Think of PCIe traffic like shipping goods:

- **TLPs** are the actual items being shipped
- **FLITs** are the standardized shipping boxes
- The PCIe link transports the boxes


## Important note

FLIT mode is especially associated with **PCIe 6.0 and later**. Earlier PCIe generations like PCIe 3.0, 4.0, and 5.0 do not rely on PCIe 6.0-style FLIT mode for normal packet transmission. #pcie_gen_3  #pcie_gen_4 #pcie_gen_5

In short:

> A PCIe FLIT is a fixed-size 256-byte flow-control unit used to carry PCIe packets, especially in PCIe 6.0+, enabling efficient transmission with error correction.

## FLIT Types
#flit  
Common FLIT types/categories are:
#flit_types #nop

| FLIT type/category | Meaning |
|---|---|
| **TLP FLIT** | A FLIT carrying one or more **Transaction Layer Packets** such as Memory Read, Memory Write, Completion, Message, etc. |
| **DLLP FLIT** | A FLIT carrying **Data Link Layer Packets**, used for flow control, ACK/NAK-related information, power management, etc. |
| **Mixed FLIT** | A FLIT that contains both **TLP** and **DLLP** information. |
| **NOP FLIT / Idle FLIT** | A FLIT sent when there is no useful packet data to transmit. It keeps the link active and aligned. |
| **Replay FLIT** | A previously transmitted FLIT resent due to error recovery. This is more of a retransmission state than a unique packet-content type. |

A simple way to think of it:

```text
PCIe FLIT
 ├── Carries TLPs
 ├── Carries DLLPs
 ├── Carries both TLPs and DLLPs
 └── Carries no useful data, called NOP/Idle FLIT
```

Important note: In PCIe, a FLIT is more like a **container**. The actual protocol information inside it is usually still **TLPs** and **DLLPs**.

## FLIT Data Structure

In **PCIe 6.0 FLIT mode**, a **FLIT** is a fixed **256-byte** structure used to transport PCIe packets.

A common PCIe 6.0 **Data FLIT** layout is:
#crc  #fec

| Field | Size | Purpose |
|---|---:|---|
| **TLP Area** | 236 bytes | Carries Transaction Layer Packets, or parts of TLPs |
| **DLP Area** | 6 bytes | Carries Data Link Layer information, flow control, ACK/NAK-type info, etc. |
| **CRC** | 8 bytes | FLIT-level error detection |
| **FEC** | 6 bytes | Forward Error Correction parity/check information |
| **Total** | **256 bytes** | One PCIe FLIT |

So the structure is roughly:

```text
+----------------------+----------------+----------+----------+
| TLP Area             | DLP Area       | CRC      | FEC      |
| 236 bytes            | 6 bytes        | 8 bytes  | 6 bytes  |
+----------------------+----------------+----------+----------+
|<------------------------ 256 bytes ------------------------->|
```

As a C-like representation:

```c
struct pcie6_flit {
    uint8_t tlp_area[236];  // TLPs or TLP fragments
    uint8_t dlp_area[6];    // Data link layer payload/control info
    uint8_t crc[8];         // FLIT CRC
    uint8_t fec[6];         // Forward error correction
};
```

## Important notes

A FLIT does **not** necessarily contain exactly one TLP.
#tlp
The **TLP Area** can contain:

```text
one TLP
multiple small TLPs
part of a large TLP
padding
```

Conceptually:

```text
PCIe Memory Write TLP
PCIe Memory Read TLP
Completion TLP
Message TLP
        ↓
packed into
        ↓
236-byte TLP area inside a 256-byte FLIT
```

### Example

If PCIe needs to send a Memory Write TLP, the packet is placed into the FLIT’s TLP area:

```text
FLIT
{
    TLP Area:
        Memory Write TLP header
        Memory Write payload data
        maybe another TLP or padding

    DLP Area:
        link-layer control/flow-control information

    CRC:
        error detection

    FEC:
        error correction
}
```


In PCIe Gen 6, the **DLP is not included inside a TLP**.  
Instead, both **TLP information** and **Data Link Layer information** are carried inside the same fixed-size **FLIT** container.

A Gen 6 FLIT is:

```text
256 bytes total

+----------------+----------+---------+---------+
| TLP area       | DLP area | CRC     | FEC     |
| 236 bytes      | 6 bytes  | 8 bytes | 6 bytes |
+----------------+----------+---------+---------+
```

So the **DLP area is beside the TLP area**, not part of the TLP itself.

---

## Why does PCIe Gen 6 put DLP information inside every FLIT?

Because PCIe Gen 6 changed to **FLIT mode** to support **PAM4 signaling**, **Forward Error Correction**, and higher error rates at 64 GT/s.

Older PCIe generations could send separate variable-size packets:

```text
TLP, DLLP, TLP, DLLP, ...
```

But Gen 6 uses a fixed-size 256-byte FLIT as the basic transfer unit:

```text
FLIT, FLIT, FLIT, FLIT, ...
```

So the Data Link Layer information needs a defined place inside that FLIT.



## Main reasons

### 1. Fixed-size FLIT format

PCIe Gen 6 transmits data in fixed 256-byte FLITs. The receiver expects this structure:

```text
[TLP area][DLP area][CRC][FEC]
```

If DLLPs were sent as separate tiny packets, they would not naturally fit the fixed FLIT-based transmission model.

So PCIe Gen 6 reserves a **6-byte DLP area** in each FLIT.



### 2. Efficient use of bandwidth

Data Link Layer messages are small but frequent.

Examples:

```text
ACK / NAK information
Flow-control credit updates
Replay control
Link management information
```

If these were sent as separate FLITs, a small control message might consume a full 256-byte FLIT, wasting bandwidth.

By embedding a small DLP area inside each FLIT, PCIe can send link-control information with minimal overhead.



### 3. Lower latency for link control

The Data Link Layer must quickly communicate things like:

```text
I received that data correctly
I detected an error
Here are updated receive credits
You may send more TLPs
```

If DLPs had to wait for separate packets or separate FLITs, link-control latency would increase.

By having a DLP area in every FLIT, PCIe Gen 6 can continuously exchange control information while also carrying TLP traffic.



### 4. Shared CRC/FEC protection

PCIe Gen 6 uses:

```text
CRC = error detection
FEC = forward error correction
```

The FLIT-level CRC and FEC protect the contents of the FLIT, including the TLP area and DLP area.

This is important because PCIe Gen 6 uses **PAM4**, which has a higher raw bit error rate than older NRZ signaling. FEC helps correct errors before they require replay.



### 5. Simpler framing at very high speed

At 64 GT/s, PCIe Gen 6 benefits from a regular, predictable structure.

Instead of mixing many variable-size packet types on the wire, the link sends a continuous stream of fixed-size FLITs:

```text
FLIT 0
FLIT 1
FLIT 2
FLIT 3
...
```

Each FLIT has a known location for:

```text
TLP payload
DLP control
CRC
FEC
```

That makes high-speed receive logic, error correction, and alignment easier.



## Important distinction

In older PCIe:

```text
TLPs and DLLPs are separate packet types.
```

In PCIe Gen 6 FLIT mode:

```text
TLPs and DLP information are carried inside a common FLIT container.
```

So the DLP is not part of the Transaction Layer Packet. It is part of the FLIT format.



 >PCIe Gen 6 includes a DLP area in each FLIT because Gen 6 uses a fixed-size FLIT-based transport. The DLP area allows link-control information such as ACK/NAK, flow control, and replay management to be carried efficiently, with low latency, and protected by the same CRC/FEC mechanism as the rest of the FLIT.



### Summary

A PCIe FLIT data structure is basically:
#dlp #crc #fec
```text
256 bytes = 236B TLP area + 6B DLP area + 8B CRC + 6B FEC
```

It is mainly used in **PCIe 6.0 and later** to support high-speed PAM4 signaling with stronger error detection and correction.