#dllp  #pcie 
In PCIe, **DLP** usually refers to a **Data Link Packet**, more officially called a **DLLP**:

```text
DLLP = Data Link Layer Packet
```

A **DLLP/DLP** is a small PCIe control packet used by the **Data Link Layer** to manage the PCIe link between two directly connected PCIe devices.

Example direct link:

```text
Root Complex  ←→  NVMe SSD
```

or through a switch:

```text
Root Complex ←→ PCIe Switch ←→ NVMe SSD
```

DLLPs are exchanged only between neighboring PCIe link partners.

---

## What does a DLLP do?

DLLPs do **not** carry normal read/write data. They are used for link control tasks such as:

```text
ACK / NAK for reliable delivery
Flow-control credit updates
Power-management signaling
Data link feature negotiation
```

---

## PCIe packet layers

PCIe commonly has these packet types:

```text
TLP   = Transaction Layer Packet
DLLP  = Data Link Layer Packet
Ordered Sets / symbols = Physical Layer signaling
```

Simplified stack:

```text
Software / driver
   ↓
Transaction Layer: TLPs
   ↓
Data Link Layer: DLLPs, sequence numbers, ACK/NAK, LCRC
   ↓
Physical Layer: electrical PCIe lanes
```

---

## TLP vs DLLP

| Item | TLP | DLLP |
|---|---|---|
| Full name | Transaction Layer Packet | Data Link Layer Packet |
| Purpose | Carries transactions/data | Controls and manages the link |
| Examples | Memory Read, Memory Write, Completion, Config Read | ACK, NAK, Flow Control Update |
| Routed through PCIe fabric? | Yes | No, local link only |
| Used by NVMe I/O? | Yes | Indirectly, for link reliability/control |

---

## Example with an NVMe SSD

When an NVMe SSD sends data to host memory, it uses PCIe **TLPs**.

Example:

```text
NVMe SSD DMA-writes data to host memory
        ↓
PCIe Memory Write TLPs
        ↓
Host receives data
```

But the PCIe link also uses **DLLPs** in the background.

Example:

```text
SSD sends TLP
Host receives it correctly
Host sends ACK DLLP
```

If an error is detected:

```text
SSD sends TLP
Host detects bad LCRC
Host sends NAK DLLP
SSD replays the TLP
```

So the normal data is in **TLPs**, while reliability/control signaling uses **DLLPs**.

---

## Common DLLP types

Examples include:

```text
ACK DLLP
NAK DLLP
InitFC DLLP
UpdateFC DLLP
Power Management DLLP
```

### ACK / NAK

Used for reliable packet delivery.

```text
ACK = received correctly
NAK = error detected, please replay
```

### Flow Control DLLPs

PCIe receivers advertise how much buffer space they have.

```text
UpdateFC = update available receive credits
```

This prevents one side from sending more TLPs than the other side can buffer.

---

## Important point

DLLPs are **link-local**.

That means a DLLP only travels across one PCIe link:

```text
Root Port ←→ Endpoint
```

or:

```text
Switch Port ←→ Endpoint
```

If a TLP goes through a PCIe switch, the switch receives and checks it on one link, then forwards it on another link. Each link has its own DLLPs, ACKs, NAKs, and flow control.

---

## Short answer

In PCIe, **DLP/DLLP** means **Data Link Layer Packet**.

It is a small PCIe control packet used for:

```text
ACK/NAK reliability
Replay handling
Flow control
Power-management/link-management signaling
```

It is different from a **TLP**, which carries actual PCIe transactions such as memory reads, memory writes, completions, and NVMe data transfers.