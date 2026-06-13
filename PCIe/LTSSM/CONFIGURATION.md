
The **Configuration state** is where PCIe decides:

1. **How many lanes will be used** (x1, x2, x4, x8, x16)
2. **Which physical lane becomes Lane 0, Lane 1, etc.**
3. **Whether lane reversal is needed**
4. **Whether both sides agree on the final link structure**

Polling proved that communication works. Configuration determines the final shape of the link.

### LTSSM Flow

```
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


# Configuration Substates

A simplified view:

```
Configuration.Linkwidth.Start
             |
             v
Configuration.Linkwidth.Accept
             |
             v
Configuration.Lanenum.Wait
             |
             v
Configuration.Lanenum.Accept
             |
             v
Configuration.Complete
             |
             v
Configuration.Idle
             |
             v
L0
```

These substates are where link width and lane numbering are negotiated.

### Step 1: Configuration.Linkwidth.Start

The Root Complex (host) leads the negotiation.
The host transmits TS1s containing:

```
Link Number = Assigned
Lane Number = PAD
```

Example:

```
Host ---> TS1(Link=1, Lane=PAD)
```

The endpoint (NVMe SSD) receives these TS1s and recognizes the proposed link number.

### Step 2: Configuration.Linkwidth.Accept

Now the host determines which lanes are actually responding.

Suppose an NVMe x4 SSD is connected.

```
Lane0 -> Responding
Lane1 -> Responding
Lane2 -> Responding
Lane3 -> Responding
```

The host decides:

```
Link Width = x4
```

If only two lanes respond:

```
Lane0 -> OK
Lane1 -> OK
Lane2 -> Failed
Lane3 -> Failed
```

The host may decide:

```
Link Width = x2
```

This is where width negotiation occurs.

### Step 3: Assign Lane Numbers

Once width is chosen, the host assigns logical lane numbers.

Example x4 link:

```
Physical Lane -> Logical Lane

Lane A -> Lane 0
Lane B -> Lane 1
Lane C -> Lane 2
Lane D -> Lane 3
```

TS1s are now transmitted with:

```
Link Number = 1
Lane Number = 0

Link Number = 1
Lane Number = 1

Link Number = 1
Lane Number = 2

Link Number = 1
Lane Number = 3
```

The endpoint learns which lane is which.

### Step 4: Configuration.Lanenum.Wait

The endpoint receives the proposed lane numbering.

It verifies:

```
Lane assignments valid?
Width valid?
Link number valid?
```

If accepted, it echoes the assignments back in TS1 ordered sets.

### Step 5: Configuration.Lanenum.Accept

Both sides now agree on:

```
Link Number
Lane Numbers
Link Width
```

Example:

```
Lane0 -> Logical 0
Lane1 -> Logical 1
Lane2 -> Logical 2
Lane3 -> Logical 3

Width = x4
```

At this point, lane mapping is complete.

### Step 6: Configuration.Complete

Both sides switch from TS1 to TS2.

```
Host <----> SSD

TS2
TS2
TS2
TS2
```

The TS2s contain the agreed link and lane numbers.

Each side verifies:

```
Received TS2 matches transmitted TS2
```

and confirms the negotiated topology.

### Step 7: Configuration.Idle

After enough TS2s are exchanged:

```
TS2 Done
```

Both sides begin transmitting Idle symbols.

```
IDL
IDL
IDL
IDL
```

This verifies the link remains stable without training sequences.

### Transition to L0

When Idle exchange succeeds:

```
Configuration.Idle
        |
        v
L0
```

Now the link is operational.

Normal traffic can begin:

```
TLPs
DLLPs
NVMe Commands
DMA Transfers
MSI/MSI-X Interrupts
```

## Example: Gen4 x4 NVMe SSD

Power-up sequence:

```
Detect
```

Receiver detected.

```
Polling
```

TS1 and TS2 exchanged.

```
Configuration
```

Host discovers:

```
4 lanes available
```

Assigns:

```
Lane0
Lane1
Lane2
Lane3
```

Negotiates:

```
Width = x4
```

Completes TS2 exchange.

```
Configuration.Idle
```

Then:

```
L0
```

The SSD can now be enumerated by the operating system.

### What Firmware Engineers Often Debug

A common failure is:

```
LTSSM = Configuration
```

and never reaches L0.

Possible causes:

- One lane not responding
- Incorrect lane numbering
- Lane reversal issue
- Width negotiation failure
- TS2 mismatch
- PHY implementation bug

Examples:

Expected:

```
Gen4 x4
```

Actual:

```
Gen4 x1
```

or

```
Stuck in Configuration.Complete
```

These problems usually indicate issues in lane negotiation rather than NVMe command processing.

# Summary

The Configuration state answers four questions:

```
1. How many lanes can we use?
2. Which lane is Lane 0, 1, 2, 3?
3. Do both sides agree?
4. Is the link stable enough for normal traffic?
```

The flow is:

```
Configuration.Linkwidth.Start
          |
          v
Configuration.Linkwidth.Accept
          |
          v
Configuration.Lanenum.Wait
          |
          v
Configuration.Lanenum.Accept
          |
          v
Configuration.Complete
          |
          v
Configuration.Idle
          |
          v
L0
```

For a typical NVMe SSD, this process ultimately establishes a **Gen1 x4 link initially**, after which the LTSSM may later enter **Recovery** to increase speed (Gen3/Gen4/Gen5) through additional training and equalization.

---

Now we move to _Config.LinkWidth.Start_. It is this, and the next state, that the viable link width or split is configured using different link numbers for each viable group of lanes. Here the upstream link (e.g., the endpoint) starts transmitting TS1s again with link and lane set to PAD. The downstream link (e.g., from root complex) start transmitting TS1s with a chosen link number and the lane number set to PAD. The upstream link responds to the receiving a minimum of two TS1s with a link number by sending back the TS1 with that link value and moves to _Config.LinkWidth.Accept_. The downstream will move to the same state when it has received to TS1s with a non-PAD link number. At this point the downstream link will transmit TS1s with assigned lane numbers whilst the upstream will initially continue to transmit TS1s with the lanes at PAD but will respond by matching the lane numbers on its TS1 transmissions (or possibly lane reversed) and then move to _Config.Lanenum.Wait_. The downstream link will move to this state on receiving TS1s with non-PAD lanes. This state is to allow for up to 1ms of time to settle errors or [[Deskew]] that could give false link width configuration. The downstream will start transmitting TS2s when it has seen two consecutive TS1s, and the upstream lanes will respond when it has received two consecutive TS2s. At this point the state is _Config.Complete_ and will move to _Config.Idle_ after receiving eight consecutive TS2s whilst sending them. The lanes start sending IDL symbols and will move to state _L_0 (LinkUp=1) after receiving eight IDL symbols and have sent at least sixteen after seeing the first IDL.

The diagram below summarizes these described steps for a normal non-split link initialization.

![[TS_during_LTSSM.png]]

To summarize these steps each link sends training sequences of a certain type and with certain values for link and lane values. When a certain number of TSs are seen, and on which lanes, the state is advanced, and configurations are set. There is a slight asymmetry in that a downstream link will switch type first to lead the upstream link into the next state. By the end of the process the link is configured for width, link number and lane assignment, with link reversal, lane inversion, and disabled scrambling where indicated. There are many variations of possible flow, such as being directed to Disabled or Loopback, or timeouts forcing the link back to Detect from Configuration states etc., which we want to describe in detail here.

A fragment of the output from the start of the link initialization of the [pcievhost](https://github.com/wyvernSemi/pcievhost) model is shown below:

![[Pasted image 20260127211200.png]]
