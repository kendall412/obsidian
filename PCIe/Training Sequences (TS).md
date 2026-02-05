In PCIe, TS1 (Training Sequence 1) and TS2 (Training Sequence 2) are special 16-symbol [[Ordered Sets]] used during link training to establish communication, detect physical link characteristics, and configure electrical equalization for high-speed data transfer. TS1 is for initial discovery, sending capabilities like link/lane numbers and data rates, while TS2 confirms those settings and finalizes the link's electrical parameters (like pre-shoot/de-emphasis) for optimal performance. 

![[TS_during_LTSSM.png]]

TS1 (Training Sequence 1)
Purpose: Initial link detection and capability announcement.

Function: Probes the physical link, advertises supported data rates, and exchanges link/lane numbers.

Content: Contains dynamic information like link/lane numbers and electrical characteristics. 

TS2 (Training Sequence 2)

Purpose: Confirm TS1 findings and finalize equalization.

Function: Confirms the link parameters found by TS1 and sets up optimal transmit equalization coefficients (cursors) for the specific link's physical path.

Content: Similar structure to TS1 but with specific fields updated for confirmation and equalization settings. 

How they work together

Initial Exchange: Devices send TS1s to "introduce" themselves and find common ground.

Equalization Setup: Based on TS1 info, the receiver requests specific equalizer settings (preset values).

Confirmation: TS2s are sent to confirm these settings and finalize the link's electrical configuration for stable, high-speed data flow. 

In essence, TS1 is the "hello, what are you?" and TS2 is the "okay, let's talk like this!" during the crucial setup phase of a PCIe link.