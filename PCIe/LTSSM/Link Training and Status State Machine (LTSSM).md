> The state of the link is defined by a <span style="color: yellow">Link Training and Status State Machine (LTSSM)</span>. LTSSM (**Link Training and Status State Machine**) is usually implemented as **hardware logic** inside the PCIe controller/PHY.From an initial state, the state machine goes through various major states (Detect, Polling, Configuration) to train and configure the link before being fully in a link-up state (L0). The initialization states also have sub-state which we will discuss shortly.

### Where LTSSM Lives

A simplified PCIe block diagram:

```
+----------------------+
| NVMe Firmware        |
+----------------------+
| NVMe Controller      |
+----------------------+
| PCIe Controller      |
|   └── LTSSM          |
+----------------------+
| PCIe PHY             |
+----------------------+
| PCIe Lanes           |
+----------------------+
```

The LTSSM is part of the PCIe hardware and runs automatically.

### Why LTSSM Is Usually Hardware

PCIe link training happens very quickly and requires real-time control of:

- Receiver detection
- TS1/TS2 transmission
- Clock recovery
- Lane alignment
- Equalization
- Speed changes

For example:

```
PERST# released
      |
      v
Detect
      |
      v
Polling
      |
      v
Configuration
      |
      v
L0
```

These transitions occur before the host can even enumerate the device.

A firmware CPU would be too slow and would complicate interoperability, so LTSSM is implemented as a [[hardware state machine]].

### What Firmware Does

Firmware typically does **not** execute the LTSSM.
Instead, firmware:
#### Monitors LTSSM

Reads status registers:

```
LTSSM State = Detect
LTSSM State = Polling
LTSSM State = L0
```

#### Responds to LTSSM Events

Example:

```
Link Up Interrupt        |        vFirmware starts initialization
```

#### Configures LTSSM Parameters

Firmware may program:

- Equalization settings
- Link speed limits
- ASPM settings
- Power-management options

But the actual state transitions are performed by hardware.

### Real NVMe SSD Example

After power-up:

```
PERST# released
```

Hardware immediately starts:

```
LTSSM Detect
LTSSM Polling
LTSSM Configuration
LTSSM L0
```

Even if the SSD firmware CPU has not yet started executing its main code. This is important because the PCIe link must come up before the host can communicate with the device.

### Hardware vs Firmware Responsibilities

#### LTSSM Hardware

```
Receiver Detection
TS1 Transmission
TS2 Transmission
Lane Alignment
Clock Recovery
Equalization
Speed Negotiation
Width Negotiation
```

#### NVMe Firmware

```
Initialize controller
Process Admin Commands
Process Read/Write Commands
FTL management
Garbage Collection
Wear Leveling
SMART data
Power Management
```

### Can Firmware Influence LTSSM?

Yes, indirectly.
Firmware may write PCIe control registers such as:

- Target link speed
- ASPM enable/disable
- Retrain Link bit
- Equalization controls

Example:

```
Firmware
    |
    v
Set Target Speed = Gen4
    |
    v
Hardware LTSSM enters Recovery
    |
    v
Retrains to Gen4
```

But firmware is not performing the training itself.

### How Engineers Often View It

Think of LTSSM like the transmission in a car:

```
Driver (Firmware)
      |
      v
Select Gear
      |
      v
Transmission Hardware
Changes Gears
```

Firmware requests actions.
LTSSM hardware executes them.

### Exceptions

Some PCIe implementations contain:

- Embedded microcontrollers
- DSP-assisted equalization engines
- Firmware-assisted PHY tuning

In these designs, firmware may assist certain training or equalization operations.

However, the core LTSSM defined by the PCIe specification is still fundamentally a **hardware state machine,** not firmware.

### Summary

**LTSSM is not typically firmware.**

It is a [[hardware state machine]] inside the PCIe controller/PHY that automatically performs:

```
Detect
Polling
Configuration
Recovery
L0
L0s
L1
L2
```

Firmware can:

- Monitor LTSSM
- Configure LTSSM behavior
- Request retraining

but the actual LTSSM transitions and protocol operations are carried out by dedicated PCIe hardware.

---
In addition, there are various powered-down states of varying degrees from L0s to L1 and L2, with L2 being all but powered off. The link can also be put into a loopback mode for test and debug, or a ‘hot reset’ state to send the link back to its initial state. The disabled state is for configured links where communications are suspended. Many of these ancillary states can be entered from the recovery state, but the main purpose of this state is to allow a configured link to change data rates, establishing lock and deskew for the new rate. Note that many of these states can be entered if directed from a higher layer, or if the link receives a particular [[Training Sequences (TS)]] ordered set where the control symbol has a particular bit set. For example, if a receiver receives two consecutive TS1 ordered sets with the Disable Link Bit asserted in the control symbol (see diagram above), the state will be forced to the Disabled state.

The diagram below shows these main states and the paths between them:

![[ltssm.png]]

From power-up, then, the main flow is from the _Detect_ state which checks what’s connected electrically and that it’s electrically idle. After this it enters the polling state where both ends start transmitting TS ordered sets and waits to receive a certain number of ordered sets from the other link. Polarity inversion is done in this state. After this, the _Configuration_ state does a multitude of things with both ends sending ordered sets moving through assigning a link number (or numbers if splitting) and the lane numbers, with lane reversal if supported. In the configuration state the received TS ordered sets may direct the next state to be _Disabled_ or _Loopback_ and, in addition, scrambling may be disabled. Deskewing will be completed by the end of this state and the link will now be ’up’ and the state enters _L_0, the normal link-up state (assuming not directed to Loopback or Disabled).

LTSSM consists of 11 top-level states:
- [[PCIe/PERST#]] (not part of LTSSM)
1. [[Detect]] - Initially detects receiver termination on the link partner.
2. [[Polling]] - Symbol lock and lane reversal detection.
3. [[Configuration]] - Negotiates link width and lane mapping.
4. [[L0]] - The fully operational state.
5. [[RECOVERY]] - Re-trains the link to change speed or fix errors.
6. L0s
7. [[L1]] - Power-saving states.
8. [[L2]] - Power-saving states.
9. [[Hot Reset]] - Resets or disables the link.
10. [[Loopback]]
11. [[Disable]] - Resets or disables the link.

The 11 top-level states can be categorized into 5 categories:
1. Link Training states (PERST => Detect => Polling => Configuration => L0)
2. Re-Training states
3. SW driven Power Management states
4. Active-State Power Management (ASPM) states
5. Other states

As mentioned before, the initialization states have sub-states, and the diagram below lists these states, what’s transmitted on those states and the condition to move to the next state.

![[link_training_steps.png]]




