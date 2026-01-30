The PCIe protocol runs serial lanes at high speed. As of version 6.0 this is 64GT/s (that is, raw bits). The SERDES that drives these serial lines at these high rates are complex and vary between manufactures and ASIC processes. The ‘Phy Interface for PCI Express’ (PIPE) was developed, by Intel, to standardize the interface between the logical protocol that we have been discussing, and the PHY sub-layer. It is not strictly part of the PCIe specification but is used so ubiquitously that I have included an overview here.

The PIPE specification conceptually splits the Physical layer into a media access layer (MAC) which includes the link training and status state machine (LTSSM), with the ordered sets and lane to lane deskew logic, a Physical Coding Sub-layer (PCS) with 8b/10b or 128b/130b codecs, RX detection and elastic buffering, and a Physical Media Attachment (PMA) with the analogue buffers and SERDES etc. The PIPE then standardizes the interface between the MAC and the PCS. The diagram below shows an overview of the PIPE signalling between the MAC and PCS:

![[Pasted image 20260127211256.png]]

The transmit data (TxData) carries the bytes for transmission. This could be wider than a byte, with 16- and 32-bit inputs allowed. The TxDataK signal indicates wither the byte is control symbol (K symbol in 8b/10b parlance). If the data interface is wider than a byte then this signal will have one wire per byte. The command signals are made up of various control signal inputs that we will discuss shortly. The data receive side mirrors the data transmit side with RxData and RxDataK signals. A set of Status signals are returned from the receiver, discussed shortly. The CLK input is implementation dependent on its specification but provides a reference for TX and RX bit-rate clock. The PCLK is the parallel data clock that all data transfers are referenced from.

The transmit command signals are summarised in the following table for PIPE version 2.00.

![[Pasted image 20260127211314.png]]

The receive status signals are summarised in the following table for PIPE version 2.00.

![[Pasted image 20260127211333.png]]

Hopefully from the tables it is easy to see how, via the PIPE interface signaling, MAC logic can control PHY state in a simple way and receive PHY status to indicate how it may transition through the LTSSM for initialisation and power down states.

The use of the PIPE standard makes development and verification much easier and allows Physical layer logic to be more easily migrated to different SERDES solutions. Usually, ASIC or IP vendors will provide IP that has this PIPE standard interface and will implement the PCS and PMA functions themselves. The vendor specific MAC logic, then, becomes more generic.