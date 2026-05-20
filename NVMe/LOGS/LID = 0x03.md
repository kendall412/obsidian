
> LID 0x03 in NVMe is the Firmware Slot Information Log Page, which reports firmware revisions stored in controller slots and indicates which firmware image is active or pending activation.

# 🔹 Firmware Slot Information Log Page

It is retrieved using the **Get Log Page Admin command** and provides information about:

- Installed firmware revisions
- Active firmware slot
- Firmware slot activation status

# 🔹 Command Context

```
Admin Opcode = 0x02 (Get Log Page)
LID          = 0x03
```


# 🔹 Purpose

NVMe controllers may support multiple firmware images stored internally.

This log page allows the host to determine:

```
- Which firmware slots contain valid images
- Which slot is currently active
- Which slot will become active after reset
```

# 🔹 Main Fields

## ✔ AFI — Active Firmware Info

Contains:

```
Bits 2:0 → currently active firmware slot
Bits 6:4 → slot to activate after reset
```


## ✔ FRS1–FRS7

Firmware Revision Strings:

```
FRS1 → firmware in slot 1
FRS2 → firmware in slot 2
...
```

Each is typically:

- 8 ASCII bytes

# 🔹 Example Output

Using nvme-cli:

```
nvme fw-log /dev/nvme0
```

Example:

```
afi  : 0x11
frs1 : 2B2QGXA7
frs2 : 2B2QGXA6
frs3 :
```

# 🔹 Decode Example

## AFI = `0x11`

Binary:

```
0001 0001
```

Decode:

```
bits 2:0 = 001 → active slot = 1
bits 6:4 = 001 → next active slot = 1
```

Meaning:

- Slot 1 is active now
- Slot 1 remains active after reset

# 🔹 Related Firmware Commands

|Opcode|Command|
|---|---|
|`0x10`|Firmware Image Download|
|`0x11`|Firmware Commit|

# 🔹 Typical Firmware Update Flow

```
1. Download firmware image
2. Commit image to slot
3. Read LID 0x03
4. Reset controller
5. New slot becomes active
```


# 🔹 Key Insight

> LID 0x03 is essentially the SSD’s firmware inventory and activation status table.