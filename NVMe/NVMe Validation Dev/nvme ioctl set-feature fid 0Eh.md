#iotcl
You're right. **FID `0x0E` is the NVMe Timestamp feature.** `0x10` is Host Controlled Thermal Management. ([GitHub](https://github.com/linux-nvme/nvme-cli/blob/master/Documentation/nvme-set-feature.txt?utm_source=chatgpt.com "nvme-cli/Documentation/nvme-set-feature.txt at master · linux-nvme/nvme-cli · GitHub"))

For **Set Features — Timestamp**, the key point is that the timestamp is supplied through a **data buffer**, not in `CDW11`. NVMe defines the timestamp as a **48-bit value in milliseconds**. After it is set, the controller updates the value as time advances. ([NVM Express](https://nvmexpress.org/changes-in-nvme-revision-1-4/?utm_source=chatgpt.com "Changes in NVMe Revision 1.4 - NVM Express"))

Here is a corrected Linux C program using `NVME_IOCTL_ADMIN_CMD`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <sys/ioctl.h>
#include <linux/nvme_ioctl.h>

#define NVME_ADMIN_SET_FEATURES  0x09
#define NVME_FID_TIMESTAMP       0x0E

int main(int argc, char *argv[])
{
    int fd;
    int ret;

    struct nvme_admin_cmd cmd;

    /*
     * Timestamp feature data structure is 8 bytes.
     *
     * Bytes 0-5:
     *     48-bit timestamp in milliseconds.
     *
     * Bytes 6-7:
     *     Reserved for Set Features.
     */
    uint8_t data[8];

    uint64_t timestamp_ms;

    if (argc != 3) {
        fprintf(stderr,
                "Usage: %s /dev/nvmeX <timestamp_ms>\n",
                argv[0]);

        fprintf(stderr,
                "Example:\n"
                "  sudo %s /dev/nvme0 123456789\n",
                argv[0]);

        return EXIT_FAILURE;
    }

    timestamp_ms = strtoull(argv[2], NULL, 0);

    /*
     * Timestamp field is only 48 bits.
     */
    if (timestamp_ms > 0xFFFFFFFFFFFFULL) {
        fprintf(stderr,
                "Error: timestamp exceeds 48-bit maximum\n");
        return EXIT_FAILURE;
    }

    /*
     * Open NVMe controller device.
     *
     * Example:
     *     /dev/nvme0
     */
    fd = open(argv[1], O_RDWR);

    if (fd < 0) {
        perror("open");
        return EXIT_FAILURE;
    }

    memset(&cmd, 0, sizeof(cmd));
    memset(data, 0, sizeof(data));

    /*
     * Put the 48-bit timestamp into bytes 0-5.
     *
     * NVMe uses little-endian byte ordering.
     */
    data[0] = (timestamp_ms >> 0)  & 0xFF;
    data[1] = (timestamp_ms >> 8)  & 0xFF;
    data[2] = (timestamp_ms >> 16) & 0xFF;
    data[3] = (timestamp_ms >> 24) & 0xFF;
    data[4] = (timestamp_ms >> 32) & 0xFF;
    data[5] = (timestamp_ms >> 40) & 0xFF;

    /*
     * NVMe Admin opcode:
     *
     * 0x09 = Set Features
     */
    cmd.opcode = NVME_ADMIN_SET_FEATURES;

    /*
     * Timestamp is controller scoped.
     */
    cmd.nsid = 0;

    /*
     * User-space address of Timestamp data structure.
     */
    cmd.addr = (uint64_t)(uintptr_t)data;

    /*
     * Timestamp data structure size.
     */
    cmd.data_len = sizeof(data);

    /*
     * CDW10:
     *
     * Bits 7:0 = FID
     * Bit 31   = SV
     *
     * FID = 0x0E
     * SV  = 0
     */
    cmd.cdw10 = NVME_FID_TIMESTAMP;

    /*
     * CDW11 is not where the timestamp value goes.
     *
     * The timestamp is carried in the data buffer.
     */
    cmd.cdw11 = 0;

    cmd.timeout_ms = 5000;

    printf("NVMe Set Features - Timestamp\n");
    printf("--------------------------------\n");

    printf("Device       : %s\n", argv[1]);
    printf("Opcode       : 0x%02X\n", NVME_ADMIN_SET_FEATURES);
    printf("FID          : 0x%02X\n", NVME_FID_TIMESTAMP);
    printf("Timestamp    : %llu ms\n",
           (unsigned long long)timestamp_ms);

    printf("\nData buffer:\n");

    for (int i = 0; i < 8; i++) {
        printf("data[%d] = 0x%02X\n", i, data[i]);
    }

    /*
     * Send NVMe Admin command.
     */
    ret = ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);

    if (ret < 0) {
        perror("NVME_IOCTL_ADMIN_CMD");
        close(fd);
        return EXIT_FAILURE;
    }

    if (ret > 0) {
        fprintf(stderr,
                "NVMe command failed, status = 0x%x\n",
                ret);

        close(fd);
        return EXIT_FAILURE;
    }

    printf("\nSet Features Timestamp succeeded.\n");
    printf("Completion DW0 result = 0x%08X\n",
           cmd.result);

    close(fd);

    return EXIT_SUCCESS;
}
```

Compile it with:

```bash
gcc -Wall -Wextra -O2 set_timestamp.c -o set_timestamp
```

Then run:

```bash
sudo ./set_timestamp /dev/nvme0 123456789
```

The actual command being built looks conceptually like this:

```text
NVMe Admin Command
──────────────────────────────────────────────

DW0
┌────────────────────────────────────────────┐
│ OPC = 0x09        Set Features             │
└────────────────────────────────────────────┘

DW1
┌────────────────────────────────────────────┐
│ NSID = 0                                   │
└────────────────────────────────────────────┘

...

DW6-DW9
┌────────────────────────────────────────────┐
│ PRP1 / data pointer                        │
│          │                                 │
│          ▼                                 │
│   Timestamp data buffer                    │
└────────────────────────────────────────────┘

DW10
31                                          0
┌───┬──────────────────────────────┬─────────┐
│SV │          Reserved            │   FID   │
│ 0 │                              │  0x0E   │
└───┴──────────────────────────────┴─────────┘

DW11
┌────────────────────────────────────────────┐
│                    0                       │
└────────────────────────────────────────────┘
```

The important distinction is:

```text
FID 0x0E
       │
       ▼
Timestamp Feature
       │
       ├── CDW10 = 0x0000000E
       │
       └── 8-byte data buffer
                 │
                 ▼
       ┌───────────────────────────┐
Byte 0 │ Timestamp bits 7:0        │
Byte 1 │ Timestamp bits 15:8       │
Byte 2 │ Timestamp bits 23:16      │
Byte 3 │ Timestamp bits 31:24      │
Byte 4 │ Timestamp bits 39:32      │
Byte 5 │ Timestamp bits 47:40      │
Byte 6 │ Reserved                  │
Byte 7 │ Reserved                  │
       └───────────────────────────┘
```

For example, suppose you set:

```text
timestamp = 0x0000_0007_5BCD_15
```

More cleanly as the 48-bit value:

```text
0x0000075BCD15
```

The memory sent to the controller is:

```text
Low address                               High address

Byte:       0     1     2     3     4     5     6     7
          +-----+-----+-----+-----+-----+-----+-----+-----+
Value:    | 15  | CD  | 5B  | 07  | 00  | 00  | 00  | 00  |
          +-----+-----+-----+-----+-----+-----+-----+-----+
             └──────── 48-bit Timestamp ────────┘
```

So:

```c
cmd.cdw10 = 0x0E;
```

selects **which feature** you are setting, while:

```c
cmd.addr = (uint64_t)(uintptr_t)data;
cmd.data_len = 8;
```

provides **the value of the Timestamp feature**.

That distinction is important because many Set Features commands put their feature-specific value in `CDW11`, but **Timestamp is one of the features that uses a data structure**. The generic `nvme set-feature` interface likewise separates FID/CDW11 values from commands that require an associated data payload. ([GitHub](https://github.com/linux-nvme/nvme-cli/blob/master/Documentation/nvme-set-feature.txt?utm_source=chatgpt.com "nvme-cli/Documentation/nvme-set-feature.txt at master · linux-nvme/nvme-cli · GitHub"))

One other detail worth knowing for validation: the host provides the timestamp in **milliseconds**, but once initialized, the controller itself advances the value. Therefore a subsequent **Get Features FID `0x0E`** should normally return a value greater than the one you set, depending on elapsed time. ([NVM Express](https://nvmexpress.org/changes-in-nvme-revision-1-4/?utm_source=chatgpt.com "Changes in NVMe Revision 1.4 - NVM Express"))