
> The **Detect state** is the very first major state of the PCIe **LTSSM (Link Training and Status State Machine)**. Its purpose is to determine: "Is there a valid PCIe receiver connected on the other end of this lane?"

Before PCIe can exchange TS1/TS2 training sequences, it must first detect that another PCIe device is physically present. The Detect state occurs immediately after:

- Power-on reset
- Hot reset
- Fundamental reset ([[PERST#]])
- Some recovery scenarios

The initial state after reset (PERST). In this state, after all the devices are powered and REFCLK is provided a Receiver circuit in each lane will determine if there is a link partner to pair with. **Each lane will begin transmitting serial data at 2.5 Gb/s (Gen 1 speed).** Detect may also be entered from a number of other LTSSM states. 

Detect Receiver Present
1. After reset or power-up ([[PERST#]]), Transmitters drive a stable voltage on the D+ and D- terminal
2. Transmitters then change the common mode voltage in a positive direction by no more than the V<sub>TX-RCV-DETECT</sub> amount of 600mV specified for all three data rates.
3. Detect logic measures the charge time:
	- Receiver is absent if the charge time is short
	- Receiver is present if the charge time is long (determined by the series capacitor and Receiver termination).

### Detect State Substates

The PCIe specification defines:

```
Detect.Quiet
      |
      v
Detect.Active
```


> In the _Detect.Quiet_ state the link waits for Electrical Idle to be broken. Electrical Idle (or not) is done with analogue circuitry, though it may be inferred with the absence of received packets or TS OSs, depending on the current state. 

### 1. Detect.Quiet

The transmitter remains electrically idle.

```
TX Driver OFF
RX Monitoring ON
```

Purpose:

- Allow signals to settle after reset
- Prevent false detection
- Ensure receiver circuitry is stable

Simplified:

```
Root Complex               NVMe SSD

TX Idle -----------------> RX

RX Waiting <------------- TX Idle
```

After a timeout, LTSSM moves to Detect.Active.

### 2. Detect.Active

> When Electrical Idle is broken, _Detect.Active_ performs a Receiver Detect by measuring for a DC impedance (40 Ω – 60Ω for PCIe 2.0) on the line. This is where actual receiver detection occurs. The transmitter performs **Receiver Detection** on each lane.

Example for a x4 NVMe link:

```
Lane 0
Lane 1
Lane 2
Lane 3
```

Each lane is tested independently.

### How Receiver Detection Works

This is an analog electrical test performed by the PHY.
The transmitter applies a small test signal to the lane.

```
TX
 |
 | Test Current
 |
 v
Lane
```

The transmitter measures the electrical response.

### Case 1: Receiver Present

```
Root Complex TX
       |
       |
       +------------------+
                          |
                      PCIe RX
```

The PCIe receiver contains a termination resistor.

Typically:

```
≈ 50 Ω to ground
```

(or equivalent differential termination)

The transmitter sees:

```
Expected Load
```

and concludes:

```
Receiver Detected
```

## Case 2: No Device Present

```
Root Complex TX
       |
       |
       +------------------+
                          X
                      Nothing
```

The transmitter sees:

```
Open Circuit
```

and concludes:

```
No Receiver
```

### What Happens After Detection?

If at least one lane detects a receiver:

```
Detect.Active      |      vPolling.Active
```

The PHY starts transmitting:

```
TS1 Ordered Sets
```

and true link training begins.

### What Happens If No Receiver Is Found?

```
Lane0 -> No Receiver
Lane1 -> No Receiver
Lane2 -> No Receiver
Lane3 -> No Receiver
```

LTSSM remains in:

```
Detect
```

and periodically retries.

This is what happens when:

- No SSD is plugged in
- A cable is disconnected
- The device is powered off
- Hardware is defective

# What the PHY Actually Does

Receiver detection is not a digital protocol exchange.

No packets are sent.

No TS1s are sent yet.

No PCIe symbols are exchanged yet.

Instead, this is a purely analog operation involving:

```
TX Driver
Receiver Termination
Current Source
Voltage Measurement
Comparator Logic
```

inside the PCIe PHY.

## LTSSM Debug Example

Suppose an NVMe SSD is not enumerating.

You inspect LTSSM:

```
LTSSM = Detect.Active
```

forever.

This suggests:

```
Receiver Detection Failed
```

Possible causes:

- SSD not powered
- Bad M.2 connector
- Broken lane
- PHY issue
- Incorrect reset sequencing
- Missing termination on receiver

# Detect State Summary

```
Detect.Quiet
      |
      v
Detect.Active
      |
      +--> Receiver Found?
               |
              Yes
               |
               v
         Polling.Active
               |
              TS1
               |
              TS2

      +--> No Receiver
               |
               v
            Retry
```

The Detect state is therefore the **electrical presence-detection phase** of PCIe LTSSM. The PHY checks each lane for the characteristic termination of a PCIe receiver. If a valid receiver is detected, the link can proceed to Polling and begin exchanging TS1/TS2 ordered sets to establish the PCIe connection.


