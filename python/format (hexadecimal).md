### Example:

```python
cmd = 'sudo sh -c \"echo 1 > /sys/bus/pci/devices/{:0>4x}\\:{:0>2x}\\:{:0>2x}.{:1x}/remove\"'.format(0, bus, device, function)
```

place holder
```python
{:0>4x}
{:0>2x}
{:0>2x}
{:1x}
```

They format numbers as hexadecimal.

```python
{:0>4x}
```

This means:
- format as hexadecimal (with `x`)
- width 4 characters
- pad with `0` on the left

```python
{:0>2x}
```

This formats `device` as 2-digit hexadecimal

```python
{:1x}
```

This formats `function` as hexadecimal with at least one character.

### Example

if:
```python
bus = 0x3b
device = 0x00
function = 0
```

then 
```bash
sudo sh -c "echo 1 > /sys/bus/pci/devices/0000\:3b\:00.0/remove"
```

the actual sysfs PCI address is:

```bash
0000:3b:00.0
```

so the path is:
```bash
/sys/bus/pci/devices/0000:3b:00.0/remove
```

