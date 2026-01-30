[https://www.dmtf.org/standards/spdm](https://www.dmtf.org/standards/spdm)

Security Protocol and Data Model (SPDM) is a security standard developed by the Distributed Management Task Force (DMTF). It enables system hardware components such as PCIe cards, NVMe drives to have their identity authenticated and their integrity verified.

The SPDM-capable components have strong cryptographic identities and can provide cryptographically signed attestations of their security state. When the server starts, SPDM-capable components are authenticated cryptographically. Measurements of their security-relevant properties are obtained to determine whether they operate at their intended state and then control is passed to the OS.

  

# libspdm is a sample implementation that follows the DMTF SPDM specifications. This is a DMTF-led effort. 
https://github.com/DMTF/libspdm

# The SPDM Responder Validator tests the protocol behavior of an SPDM Responder device to validate that it conforms to the SPDM specification. This is a DMTF-led effort. 
SPDM-Responder-Validator/doc/1.Version.md at d4b33d4b5a2a63cd25c136d8ad6b18fd14b8b196 · DMTF/SPDM-Responder-Validator · GitHub 