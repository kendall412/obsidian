I said earlier that the serial lines were [AC coupled](https://www.keysight.com/used/us/en/knowledge/glossary/oscilloscopes/what-is-ac-coupling). The first layer of protocol that we will look at is the encoding of bytes. Data is serialized from bytes, but the bytes are encoded into DC free signals using one of two encoding schemes:

- [8b/10b encoding](https://en.wikipedia.org/wiki/8b/10b_encoding) (version 2.1 and earlier)
- [128b/130b encoding](https://en.wikipedia.org/wiki/64b/66b_encoding) (version 3.0 onwards)

Both of these encodings allow three things. Firstly, they allow minimal localized DC components with the average signal DC free. Secondly, they allow clock recovery from the data with a guaranteed transition rate. Thirdly they allow differentiation between control information and data. We will not look at the details of the two encodings here.

Having encoded the bytes for each lane, the SERDES (serializer-deserializer) will turn this into a bit stream and send out least-significant bit first. Other than when the lane is off there will always be a code transmitted, with one of the codes being IDLE if there is nothing to send. At the receiving end, the SERDES will decode the encoded values and produce a stream of bytes.

Within the 8b/10b encoding are control symbols, as mentioned before, called K symbols, and for PCIe these are encoded to have the following meanings.

![[8b_10b_k_symbols.png]]

For 128b/130b encoding of the two control bits determine whether the following 16 bytes are an ordered set (10b) or data (01b), rather than a K symbol. When an ordered set, the first symbol determines the type of ordered set. Thus, the 10b control bits act like a COM symbol, and the next symbol gives the value, whereas 01b control bits have symbol 0 encode the various other token types. More details are given in the next section.




**[https://www.01signal.com/using-ip/mgt/encodings/](https://www.01signal.com/using-ip/mgt/encodings/)**