
#nvme-cli 
To list the NVMe log pages supported by a drive, use:
```
sudo nvme supported-log-pages /dev/nvme0 -H
```

or JSON output:
```
sudo nvme supported-log-pages /dev/nvme0 -o json
```

If your `nvme-cli` does not have that command, you can read Log Page `0x00` directly:
```
sudo nvme get-log /dev/nvme0 --log-id=0x00 --log-len=1024 --raw-binary | hexdump -C
```

## LID/Log Table (NVMe Base Specification 2.2 3.1.3.5 Log Page Support Requirements pg 45)
|LID|Logs|
|:-:|---|
|0x00|Supported Log Pages|
|0x01|Error Information Log|
|0x02|SMART / Health Information Log|
|0x03|Firmware Slot Information Log|
|0x04|Changed Namespace List Log|
|0x05|Commands Supported and Effects Log|
|0x06|Device Self-test Log|
|0x07|Telemetry Host-Initiated Log|
|0x08|Telemetry Controller-Initiated Log|
|0x09|Endurance Group Information Log|
|0x0A|Predictable Latency per NVM Set Log|
|0x0B|Predictable Latency Event Aggregate Log|
|0x0C|Asymmetric Namespace Access, ANA, Log|
|0x0D|Persistent Event Log|
|0x0E|LBA Status Information Log|
|0x0F|Endurance Group Event Aggregate Log|
|0x10|Media Unit Status Log|
|0x11|Supported Capacity Configuration List Log|
|0x12|Feature Identifiers Supported and Effects Log|
|0x13|NVMe-MI Commands Supported and Effects Log|
|0x14|Command and Feature Lockdown Log|
|0x15|Boot Partition Log|
|0x16|Rotational Media Information Log|
|0x70|Discovery Log Page, mainly NVMe-oF|
|0x80|Reservation Notification Log|
|0x81|Sanitize Status Log|
|0xBF|Changed Zone List Log, Zoned Namespace/ZNS|
|0xC0-0xFF|Vendor-specific Logs|
