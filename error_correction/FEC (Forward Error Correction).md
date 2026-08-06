
>**FEC (Forward Error Correction)** is a way for the receiver to **detect and correct some bit errors without asking the transmitter to resend the data**. In PCIe, this is especially important in **PCIe 6.0**, because PCIe 6.0 uses [[PAM4 signaling]], which carries more bits per symbol but is more sensitive to noise than older NRZ signaling.

## Basic idea

The transmitter sends:

```text
original data + extra check/parity information
```

The receiver uses the extra information to determine whether some bits were corrupted and, if possible, correct them.

So instead of:

```text
receive bad data → request retry
```

FEC allows:

```text
receive slightly bad data → correct it locally → continue
```

## Simple analogy

Imagine sending this number:

```text
1234
```

But you also send some extra checking information, such as:

```text
sum = 10
```

The receiver gets:

```text
1235, sum = 10
```

It knows something is wrong because:

```text
1 + 2 + 3 + 5 = 11
```

In real FEC, the math is much more advanced, so the receiver can often determine **which bit or bits are wrong** and fix them.

## Why PCIe needs FEC

PCIe 6.0 runs at:

```text
64 GT/s
```

To reach that speed, it uses **PAM4**, where each signal level carries 2 bits.

Compared with older signaling, PAM4 has smaller voltage margins, so it is more vulnerable to:

- electrical noise
- signal reflections
- crosstalk
- channel loss
- jitter

FEC helps maintain a very low error rate even at these high speeds.


## FEC vs [[CRC (Cyclic Redundancy Check)]]

| Feature | FEC | CRC |
|---|---|---|
| Full name | Forward Error Correction | Cyclic Redundancy Check |
| Purpose | Correct errors | Detect errors |
| Can fix bad bits? | Yes, within limits | No |
| Avoids retransmission? | Often yes | No, usually triggers retry/replay |
| Used in PCIe 6.0 FLIT? | Yes | Yes |

So:

```text
FEC tries to fix errors.
CRC checks whether the final data is valid.
```

---

## In PCIe FLITs

A PCIe 6.0 FLIT has FEC bits included. Conceptually:

```text
+-------------------+------------------+
| FLIT data          | FEC parity/check |
+-------------------+------------------+
```

When the receiver gets the FLIT:

1. It uses **FEC** to correct small errors.
2. It checks the **CRC** to verify the FLIT is correct.
3. If the CRC still fails, PCIe may use error handling/retry mechanisms.

---

## Short summary

**Forward Error Correction** means:

> Add extra parity/check bits to transmitted data so the receiver can correct limited errors by itself, without needing retransmission.

In PCIe 6.0, FEC is a key feature that makes very high-speed PAM4 links reliable.