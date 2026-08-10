
> In a PCIe device, a Physical Function (PF) is a full, real PCIe function implemented by the hardware and visible to the operating system during PCIe enumeration. A Physical Function (PF) in PCIe is not necessarily a separate physical device like a separate card or chip. It is a real hardware-backed PCIe function exposed by a physical PCIe device.

A PF has its own PCI configuration space, Base Address Registers, interrupts, and device identity. The OS can load a driver for it just like for any normal PCIe device.

```
Physical PCIe card/device
└── PCIe Physical Function 0
└── PCIe Physical Function 1
└── PCIe Physical Function 2
```

|Term|Meaning|
|---|---|
|Physical Device|The actual HW card/chip/SSD|
|Physical Function|A real PCIe function implemented by that HW|
|Virtual Function|A lightweight PCIe function created for virutalization, by SR-IOV|

### Examples:

- A simple NVMe SSD usually exposes one Physical Function
- A multi-port network card may expose one PF per port
- Some enterprise NVMe or accelerator devices expose multiple PFs

In PCIe, a device is addressed like this:
```
Bus : Device : Function
```
```
0000:5e:00.0
```
Here:

- `0000` = PCI domain
- `5e` = bus
- `00` = device
- `.0` = function number

If the same PCIe device exposes another PF, you might see:
```
0000:5e:00.0
0000:5e:00.1
```
Those are two different physical functions on the same PCIe device.

### A PF is different from a Virtual Function (VF).

With SR-IOV:

- PF = full hardware function, controlled by the host/hypervisor
- VF = lightweight virtual PCIe function created by the PF for virtual machines or containers

For example, a network card might show:
```
PF: 0000:5e:00.0
VF: 0000:5e:10.0
VF: 0000:5e:10.1
```

The PF driver manages the hardware and creates/configures VFs.

In short: A Physical Function is a real PCIe hardware function that the OS can discover, configure, and use directly. 