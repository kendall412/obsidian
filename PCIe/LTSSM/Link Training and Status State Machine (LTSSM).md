> The state of the link is defined by a <span style="color: yellow">Link Training and Status State Machine (LTSSM)</span>. From an initial state, the state machine goes through various major states (Detect, Polling, Configuration) to train and configure the link before being fully in a link-up state (L0). The initialization states also have sub-state which we will discuss shortly.

In addition, there are various powered-down states of varying degrees from L0s to L1 and L2, with L2 being all but powered off. The link can also be put into a loopback mode for test and debug, or a ‘hot reset’ state to send the link back to its initial state. The disabled state is for configured links where communications are suspended. Many of these ancillary states can be entered from the recovery state, but the main purpose of this state is to allow a configured link to change data rates, establishing lock and deskew for the new rate. Note that many of these states can be entered if directed from a higher layer, or if the link receives a particular [[Training Sequences (TS)]] ordered set where the control symbol has a particular bit set. For example, if a receiver receives two consecutive TS1 ordered sets with the Disable Link Bit asserted in the control symbol (see diagram above), the state will be forced to the Disabled state.

The diagram below shows these main states and the paths between them:

![[ltssm.png]]

From power-up, then, the main flow is from the _Detect_ state which checks what’s connected electrically and that it’s electrically idle. After this it enters the polling state where both ends start transmitting TS ordered sets and waits to receive a certain number of ordered sets from the other link. Polarity inversion is done in this state. After this, the _Configuration_ state does a multitude of things with both ends sending ordered sets moving through assigning a link number (or numbers if splitting) and the lane numbers, with lane reversal if supported. In the configuration state the received TS ordered sets may direct the next state to be _Disabled_ or _Loopback_ and, in addition, scrambling may be disabled. Deskewing will be completed by the end of this state and the link will now be ’up’ and the state enters _L_0, the normal link-up state (assuming not directed to Loopback or Disabled).

LTSSM consists of 11 top-level states:
1. Detect - Initially detects receiver termination on the link partner.
2. Polling - Symbol lock and lane reversal detection.
3. Configuration - Negotiates link width and lane mapping.
4. [[L0]] - The fully operational state.
5. [[Recovery]] - Re-trains the link to change speed or fix errors.
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

# PRE-DETECT
Control signals:
[[PERST]]# - PERST (fundamental reset) is performed. It is held low until all power rails are stable. Low to high signals indicate the beginning of link initialization.
WAKE#
CLKREQ#

REFCLK (100MHz) is a prerequisite.

# 1. [[DETECT]]

# 2. [[POLLING]]

# 3. [[CONFIGURATION]]

# 4. [[L0]]

