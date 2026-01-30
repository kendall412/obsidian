The basic data unit of PCIe is a byte which will be encoded prior to serialisation. Before this encoding, though, the data bytes are scrambled on a per lane basis. This is done with a 16-bit linear feedback shift register or an equivalent. The polynomial used for PCIe 2.0 and earlier is G(x) = x16+x5+x4+x3+1, whilst for 3.0 it is G(x) = x23+x21+x16+x8+x5+x2+1. The scrambling can be disabled during initialization, but this is normally for test and debug purposes.

To keep the scrambling synchronized across multiple lanes the [LSFR](https://www.geeksforgeeks.org/digital-logic/linear-feedback-shift-registers-lfsr/) is reset to 0xffff when a COM symbol is processed (see below). Also, it is not advanced when a SKP symbol is encountered since these may be deleted in some lanes for alignment (more later). The K symbols are not scrambled and also data symbols within a training sequence.

Data scrambling: To improve electrical characteristics of a link, data is scrambled. XORing of data stream with a pattern generator by LFSR (Linear Feedback Shift Reg). On Tx side data is scrambled and on Rx side data is de-scrambled.  
Scrambling is performed by serially XORing the 8bit(D0-D7) character with 16bit(D0-D15) output of LFSR. D15 is XORed with D0 of the data to be processed.  
G(x)=x^16 + x^5 + x^4 + x^3 +1  

Rules of data scrambling:  
·      COM symbols initiate the LFSR.  
·      All special symbols (K codes) are not scrambled.  
·      Scrambled is always enabled in Detect by default and disabled at the End of configuration.  
·      Scrambled is not enabled at loopback mode.  
·      All data symbols(D codes) except those within Ordered sets (TS1, TS2, EIEOS) are scrambled.

![[Pasted image 20260127210602.png]]