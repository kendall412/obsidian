In PCIe, an Ordered Set (OS) is a special, short control packet used for link management, like synchronization, training, and flow control, rather than sending user data; they ensure the physical link stays aligned, manage speed changes (like entering/exiting low power states), and signal link status, with common types including TS1/TS2 (Training Sequences), EIEOS (Electrical Idle Exit), and SKP (Skip) Ordered Sets, acting as the "punctuation" of the physical layer. 

Purpose of Ordered Sets

Link Training & Initialization: They help devices discover each other, establish a connection, and calibrate link parameters (deskewing, timing).

Flow Control & Synchronization: SKP Ordered Sets, for instance, are used to adjust for clock frequency differences and maintain data alignment (elastic buffering).

Power Management: They signal transitions into and out of low-power states, like L0s (Electrical Idle).

Control Signals: They carry essential physical layer (PHY) control information, helping manage the link's state machine. 

Key Types of Ordered Sets

TS1/TS2 (Training Sequence 1/2): Used extensively during link training for alignment and initial communication.

EIEOS (Electrical Idle Exit Ordered Set): Signals the end of an electrical idle state, allowing the link to wake up.

EIOS (Electrical Idle Ordered Set): Used to indicate entry into electrical idle.

SKP (Skip): Inserted periodically to compensate for clock drift, allowing receivers to adjust their elastic buffers.

Control Skip: A special type for finer control, especially in newer PCIe versions like PCIe 6.0. 

How They Work

Framing: A special control bit (e.g., 10b in some encodings) marks a block of symbols as an Ordered Set versus regular data.

Symbol Structure: Each OS has a specific structure (e.g., 16 symbols for TS1/TS2) where the first symbol identifies the type (e.g., TS1, TS2).

Periodic Insertion: SKP OSes are scheduled at set intervals (e.g., every 1538 symbol times) if no other packet is ready to send, ensuring continuous link maintenance. 

In essence, while data packets (TLPs) carry the actual information, Ordered Sets are the crucial background communication that keeps the PCIe link healthy and functioning at high speeds.

So now we know how to send bytes out over the lanes, some of which are scrambled, all of which are encoded, and some of the encoded symbols are control symbols. Using these encodings, the protocol can encode blocks of data either within a lane, or frame packets across the available lanes. Within the lanes are ‘ordered sets’ which are used during initialisation and link training. (We will deal with the link training in a separate section as it is a large subject.) The diagram below shows the ordered sets for PCIe versions up to 2.1:

![[Pasted image 20260127210925.png]]

The training sequence OS, as we shall see in a following section, are sent between connected lanes advertising the PCIe versions supported and link and lane numbers. The training will start at the 1.x speeds and then, at the appropriate point switch to the highest rate supported by both ends. The link number is used to allow for possible splitting of a link. For example, if a downstream link is x8, and connected to two x4 upstream links, the link numbers will be N and N+1. The lane number is used to allow the reversal of lanes in a link when lane 0 connects to a lane N-1, and lane N-1 connects to 0. This can be reversed, meaning the internal logic design still sends the same data out on its original, unreversed, lane numbers. By sending the lane number, the receiving link knows that this has happened. This may seem a strange feature, but this may occur due to, say, layout constraints on a PCB and reassigning lane electrically helps in this regard. In addition, the training will also detect lanes that have their differential pairs wired up incorrectly when a receiver may see inverted TSX symbols in the training sequences on one or more lanes and will, at the appropriate point in the initialization, invert the data.

The Electrical idle OS is sent on active lines by a transmitter immediately before entering an electrical idle state (which is also the normal initial state).

The Fast Training OS is sent when moving from a particular power saving state (L0s) back to L0 (normal operation) to allow a quick bit and symbol lock procedure without a full link recovery and training. The number of fast training OS blocks sent was configured during link initialisation with the N_FTS value in the training sequence OSs.

### PCIe 3.0+ Ordered Sets

For specifications beyond 2.1 a training sequence consists of a 130-bit code with 2 bits of control and 16 bytes. The leading control determines which type of data follows, with 01b a data block (across lanes) and 10b an ordered set. This takes the place of a comma for the ordered sets. The following 16 bytes define which ordered set is present

- **Training Sequence Ordered Set**: First symbol of 1Eh (TS1) or2Dh (TS2), followed by the same information fields as above (though PAD is encoded as F7h). Symbols 6 to 9 replace TSX values with addition configuration values such as equalization and other electrical settings.
- **Electrical Idle Ordered Set**: All symbols 66h
- **Fast Training Ordered Set**: a sequence of: 55h, 47h, 4Eh, C7h, CCh, C6h, C9h, 25h, 6Eh, ECh, 88h, 7Fh, 80h, 8Dh, 8Bh, 8Eh
- **Skip Ordered Set**: 12 symbols of AAh, a skip end symbol of E1h, last 3 symbols status and LSFR values. Not that the first 12 symbols can vary in length since symbols may be added or deleted for lane-to-lane deskew.