#pcie #dlp #dllp
In a **PCIe Gen 6 FLIT**, the **DLP area** is the fixed part of the FLIT reserved for **Data Link Layer information**.

A PCIe Gen 6 FLIT is **256 bytes**:
#flit #pcie_gen_6 
```text
+----------------------+----------+----------+----------+
| TLP area             | DLP area | CRC      | FEC      |
| 236 bytes            | 6 bytes  | 8 bytes  | 6 bytes  |
+----------------------+----------+----------+----------+
```

So the **DLP area is 6 bytes** inside every 256-byte FLIT.

---

## What does the DLP area carry?

The DLP area carries link-local **Data Link Layer Packets / payloads**, similar in purpose to older PCIe **DLLPs**.

It is used for things like:

```text
ACK / NAK information
Flow-control credit updates
Replay/retry control
Power-management/link-management information
Other data-link control messages
```

It does **not** carry normal user data or NVMe data. Normal PCIe transactions such as Memory Reads, Memory Writes, Completions, and NVMe DMA traffic go in the **TLP area**.

---

## Gen 6 FLIT layout example

```text
256-byte PCIe Gen 6 FLIT

Bytes 0   - 235 : TLP area
Bytes 236 - 241 : DLP area
Bytes 242 - 249 : CRC
Bytes 250 - 255 : FEC
```

Or visually:

```text
[ TLP payload / TLP fragments ][ DLP ][ CRC ][ FEC ]
        236 B                  6 B    8 B    6 B
```

---

## TLP area vs DLP area

| Field | Size | Purpose |
|---|---:|---|
| TLP area | 236 B | Carries Transaction Layer Packets |
| DLP area | 6 B | Carries Data Link Layer control information |
| CRC | 8 B | Error detection |
| FEC | 6 B | Forward error correction |

---

## Important point

The **DLP area is link-local**.

That means it applies only between two directly connected PCIe ports, for example:

```text
Root Port  ←→  Endpoint
```

or:

```text
Switch Port  ←→  Endpoint
```

If a TLP passes through a PCIe switch, the TLP may continue through the fabric, but the **DLP information is consumed and regenerated per link**.

---

## Simple summary

In PCIe Gen 6 FLIT mode:

```text
DLP area = 6-byte field in each 256-byte FLIT
```

It is used by the PCIe **Data Link Layer** for control functions such as flow control, ACK/NAK, retry/replay, and link management. It is separate from the **TLP area**, which carries actual PCIe transactions and data.