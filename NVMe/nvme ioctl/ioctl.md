
`ioctl` stands for **Input/Output Control**. It is a Unix/Linux **system call** that lets a user-space program send a **special control request** to a kernel driver.

```
#include <sys/ioctl.h>

int ioctl(int fd, unsigned long request, ...);
```

The basic idea is:

```
User program
    │
    │ ioctl()
    ▼
Linux kernel
    │
    ▼
Device driver
    │
    ▼
Hardware
```

For NVMe:

```
Your C program
    │
    │ ioctl()
    ▼
Linux NVMe driver
    │
    ├── Builds NVMe command
    ├── Manages DMA
    ├── Submits command to SQ
    ├── Rings doorbell
    └── Processes CQ completion
    │
    ▼
NVMe SSD
```

## 1. Why use `ioctl()`?

Normal system calls such as:

```
read();
write();
```

are designed for ordinary data transfer.

For example:

```
read(fd, buffer, 4096);
```

means: Read 4096 bytes. But many devices have operations that do not fit naturally into `read()` or `write()`.

For an NVMe device, you may want to issue:

- Identify Controller
- Identify Namespace
- Get Log Page
- Get Features
- Set Features
- Device Self-test
- Vendor-specific commands

`ioctl()` provides a way to ask the device driver to perform these specialized operations.

## 2. The three main arguments

A typical call looks like:

```
ioctl(fd, request, &data);
```

Breaking it down:

```
ioctl(fd, request, &data)
       │      │       │
       │      │       └── Information for the request
       │      │
       │      └────────── Type of operation
       │
       └───────────────── Which open device
```

### Argument 1: `fd`

`fd` is the **file descriptor** returned by `open()`.

Example:

```
int fd = open("/dev/nvme0", O_RDONLY);
```

Suppose Linux returns:

```
fd = 3
```

Now:

```
3
│
▼
/dev/nvme0
│
▼
Linux NVMe driver
│
▼
NVMe controller
```

you use that `fd` with `ioctl()`:

```
ioctl(fd, ...);
```

### Argument 2: `request`

The request tells the kernel driver **what kind of operation you want**.

For NVMe:

```
NVME_IOCTL_ADMIN_CMD
```

means:

> Submit an NVMe Admin passthrough command.

For example:

```
ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);
```

For an NVMe I/O passthrough command:

```
ioctl(fd, NVME_IOCTL_IO_CMD, &cmd);
```

These constants come from:

```
#include <linux/nvme_ioctl.h>
```

### Argument 3: data or command structure

The third argument usually points to a structure containing the details of the operation.

For NVMe:

```
struct nvme_admin_cmd cmd;
```

You fill in the command:

```
cmd.opcode = 0x06;
cmd.nsid = 0;
cmd.addr = (uint64_t)buffer;
cmd.data_len = 4096;
cmd.cdw10 = 1;
```

Then pass its address:

```
ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);
```

## 3. General pattern for using `ioctl()`

Most `ioctl()` programs follow this pattern:

```
#include <fcntl.h>
#include <stdio.h>
#include <sys/ioctl.h>
#include <unistd.h>

int main(void)
{
    // 1. Open the device
    int fd = open("/dev/device", O_RDWR);

    if (fd < 0) {
        perror("open");
        return 1;
    }

    // 2. Prepare request data
    // ...

    // 3. Send ioctl request
    int ret = ioctl(fd, SOME_IOCTL_REQUEST, &data);

    if (ret < 0) {
        perror("ioctl");
        close(fd);
        return 1;
    }

    // 4. Process result
    // ...

    // 5. Close the device
    close(fd);

    return 0;
}
```

The exact `request` and `data` structure depend on the driver.

That is important:

> `ioctl()` itself does not define what operations are available. The device driver defines the supported ioctl requests and argument structures.

## 4. Simple non-NVMe example

Here is an example that gets the terminal window size:

```
#include <stdio.h>
#include <sys/ioctl.h>
#include <unistd.h>

int main(void)
{
    struct winsize size;

    int ret = ioctl(STDOUT_FILENO, TIOCGWINSZ, &size);

    if (ret < 0) {
        perror("ioctl");
        return 1;
    }

    printf("Rows: %d\n", size.ws_row);
    printf("Columns: %d\n", size.ws_col);

    return 0;
}
```

The call:

```
ioctl(STDOUT_FILENO, TIOCGWINSZ, &size);
```


means:

```
Which device?
    │
    └── STDOUT_FILENO

What request?
    │
    └── TIOCGWINSZ
        "Get terminal window size"

Where should the result go?
    │
    └── &size
```

The kernel fills:

```
struct winsize size;
```

with information.

## 5. Using `ioctl()` with NVMe

For NVMe programming on Linux, include:

```
#include <linux/nvme_ioctl.h>
#include <sys/ioctl.h>
```

A common structure is:

```
struct nvme_admin_cmd
```

A simplified workflow is:

```
1. Open /dev/nvme0
        │
        ▼
2. Allocate data buffer
        │
        ▼
3. Create nvme_admin_cmd
        │
        ▼
4. Fill command fields
        │
        ▼
5. Call ioctl()
        │
        ▼
6. Linux NVMe driver submits command
        │
        ▼
7. NVMe controller executes command
        │
        ▼
8. ioctl() returns
        │
        ▼
9. Read returned data
```

## 6. Example: NVMe Identify Controller

The NVMe **Identify** Admin command has:

```
Opcode = 0x06
```

For **Identify Controller**:

```
CNS = 0x01
```

The controller returns:

```
4096 bytes
```

### Complete example

```
#include <errno.h>
#include <fcntl.h>
#include <linux/nvme_ioctl.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ioctl.h>
#include <unistd.h>

int main(void)
{
    const char *device = "/dev/nvme0";

    /*
     * Step 1:
     * Open the NVMe controller device.
     */
    int fd = open(device, O_RDONLY);

    if (fd < 0) {
        perror("open");
        return 1;
    }

    /*
     * Step 2:
     * Allocate a 4096-byte buffer.
     *
     * The Identify command returns 4096 bytes.
     */
    void *buffer = NULL;

    if (posix_memalign(&buffer, 4096, 4096) != 0) {
        fprintf(stderr, "Memory allocation failed\n");
        close(fd);
        return 1;
    }

    memset(buffer, 0, 4096);

    /*
     * Step 3:
     * Create the NVMe Admin command structure.
     */
    struct nvme_admin_cmd cmd;

    memset(&cmd, 0, sizeof(cmd));

    /*
     * Step 4:
     * Fill in the command fields.
     */

    // NVMe Identify opcode
    cmd.opcode = 0x06;

    // Identify Controller uses NSID = 0
    cmd.nsid = 0;

    // Address of the data buffer
    cmd.addr = (uint64_t)(uintptr_t)buffer;

    // Identify data structure size
    cmd.data_len = 4096;

    // CDW10 bits 7:0 = CNS
    // CNS = 1 means Identify Controller
    cmd.cdw10 = 1;

    /*
     * Step 5:
     * Submit the command to the Linux NVMe driver.
     */
    int ret = ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);

    /*
     * Step 6:
     * Check the result.
     */
    if (ret < 0) {
        perror("NVME_IOCTL_ADMIN_CMD");

        free(buffer);
        close(fd);
        return 1;
    }

    if (ret > 0) {
        printf("NVMe command returned status: 0x%x\n", ret);

        free(buffer);
        close(fd);
        return 1;
    }

    printf("Identify Controller succeeded\n");

    /*
     * Step 7:
     * Examine returned data.
     */
    unsigned char *data = buffer;

    printf("First 64 bytes:\n");

    for (int i = 0; i < 64; i++) {
        printf("%02x ", data[i]);

        if ((i + 1) % 16 == 0)
            printf("\n");
    }

    /*
     * Step 8:
     * Clean up.
     */
    free(buffer);
    close(fd);

    return 0;
}
```

Compile:

```
gcc -Wall -O2 nvme_identify.c -o nvme_identify
```

Run:

```
sudo ./nvme_identify
```

## 7. What exactly happens at the `ioctl()` line?

The key line is:

```
int ret = ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);
```

Let's break it down.

```
fd
│
└── /dev/nvme0
    "Which NVMe controller?"
```

```
NVME_IOCTL_ADMIN_CMD
│
└── "I want to submit an NVMe Admin command"
```

```
&cmd
│
└── "Here are the command parameters"
```

Together:

```
ioctl(
    fd,                       ← Which device?
    NVME_IOCTL_ADMIN_CMD,     ← What operation?
    &cmd                      ← What parameters?
);
```

In plain English:

> "Ask the Linux NVMe driver controlling `/dev/nvme0` to submit the Admin command described by `cmd`."

## 8. What happens inside Linux?

```
USER SPACE
══════════════════════════════════════════

Your program

struct nvme_admin_cmd
┌───────────────────────────┐
│ opcode = 0x06             │
│ nsid = 0                  │
│ addr = buffer             │
│ data_len = 4096           │
│ cdw10 = 1                 │
└───────────────────────────┘
             │
             │
             ▼
ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd)


══════════ SYSTEM CALL BOUNDARY ══════════


KERNEL SPACE
══════════════════════════════════════════

Linux ioctl handling
             │
             ▼
Linux NVMe driver
             │
             ├── Validates request
             │
             ├── Copies command parameters
             │
             ├── Maps user buffer for DMA
             │
             ├── Builds NVMe command
             │
             ├── Allocates command ID
             │
             ├── Places command in SQ
             │
             └── Rings SQ doorbell
             │
             ▼
         NVMe SSD
             │
             ├── Reads command
             ├── Executes Identify
             ├── DMA writes 4096 bytes
             └── Posts CQ entry
             │
             ▼
Linux NVMe driver
             │
             ├── Processes completion
             ├── Unmaps DMA
             └── Returns status
             │

══════════ SYSTEM CALL BOUNDARY ══════════

             ▼
        ioctl() returns


USER SPACE
══════════════════════════════════════════

Your program reads:

buffer[0 ... 4095]
```

## 9. Does `ioctl()` copy the entire 4096-byte buffer into the kernel?

Not necessarily in the simple sense of:

```
User buffer
    │
    │ memcpy()
    ▼
Kernel buffer
```

For NVMe data transfer, the kernel can pin/map the application's memory for DMA.

Conceptually:

```
User virtual address
       │
       ▼
Kernel memory management
       │
       ├── Pin pages
       ├── Create DMA mapping
       └── Build PRP/SGL representation
       │
       ▼
NVMe Controller
       │
       │ DMA
       ▼
Physical memory
```

Your program supplies:

```
cmd.addr = (uint64_t)(uintptr_t)buffer;
```

This is a **user-space virtual address**. The NVMe controller does **not simply use that virtual address as PRP1**.

Instead:

```
User virtual address
       │
       ▼
Linux NVMe driver
       │
       ▼
DMA mapping
       │
       ▼
PRP/SGL
       │
       ▼
NVMe controller
```

## 10. Relationship between `nvme_admin_cmd` and the NVMe SQE

The user-space structure may look conceptually like:

```
struct nvme_admin_cmd

┌─────────────────────┐
│ opcode              │
│ flags               │
│ nsid                │
│ metadata            │
│ addr                │
│ metadata_len        │
│ data_len            │
│ cdw10               │
│ cdw11               │
│ cdw12               │
│ cdw13               │
│ cdw14               │
│ cdw15               │
│ timeout_ms          │
│ result              │
└─────────────────────┘
```

The kernel converts this into the actual **64-byte NVMe command**:

```
NVMe Submission Queue Entry
64 bytes

Byte 0
┌────────────────────────────────┐
│ CDW0: OPC, FUSE, PSDT, CID     │
├────────────────────────────────┤
│ CDW1: NSID                     │
├────────────────────────────────┤
│ CDW2                           │
├────────────────────────────────┤
│ CDW3                           │
├────────────────────────────────┤
│ CDW4-5: MPTR                   │
├────────────────────────────────┤
│ CDW6-7: DPTR / PRP1            │
├────────────────────────────────┤
│ CDW8-9: DPTR / PRP2            │
├────────────────────────────────┤
│ CDW10                          │
├────────────────────────────────┤
│ CDW11                          │
├────────────────────────────────┤
│ CDW12                          │
├────────────────────────────────┤
│ CDW13                          │
├────────────────────────────────┤
│ CDW14                          │
├────────────────────────────────┤
│ CDW15                          │
└────────────────────────────────┘
Byte 63
```

or Identify Controller:

```
User program                     Hardware SQE
──────────────────────────────────────────────

cmd.opcode = 0x06       ───────► OPC = 0x06

cmd.nsid = 0            ───────► NSID = 0

cmd.addr = buffer       ───────► DMA mapping
                                 │
                                 ▼
                              PRP/SGL

cmd.cdw10 = 1           ───────► CDW10.CNS = 1
```

## 11. `ioctl()` is synchronous in this example

When you call:

```
ret = ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);
```

your thread normally waits:

```
ioctl()
   │
   ├── Submit NVMe command
   │
   ├── Wait...
   │
   ├── NVMe command completes
   │
   └── Return
```

So:

```
printf("before\n");

ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);

printf("after\n");
```

typically behaves like:

```
before

[command executes]

after
```

The application does not normally receive control back until the passthrough command completes or fails.

## 12. How to know which `ioctl` request to use

You must consult the interface for the particular driver.

For NVMe, Linux defines interfaces such as:

```
NVME_IOCTL_ADMIN_CMD
NVME_IOCTL_IO_CMD
NVME_IOCTL_ID
NVME_IOCTL_RESET
NVME_IOCTL_SUBSYS_RESET
```

The relevant header is:

```
#include <linux/nvme_ioctl.h>
```

You can inspect it on Linux:

```
cat /usr/include/linux/nvme_ioctl.h
```

Or:

```
grep -n "NVME_IOCTL" /usr/include/linux/nvme_ioctl.h
```

The header will contain definitions similar to:

```
#define NVME_IOCTL_ID
#define NVME_IOCTL_ADMIN_CMD
#define NVME_IOCTL_IO_CMD
```

## 13. The most important distinction

`ioctl()` is **not an NVMe function**.

It is a general Linux system call.

Many different drivers use it:

```
                    ioctl()
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
   NVMe driver    Terminal driver   Network driver
        │              │              │
        ▼              ▼              ▼
  NVMe commands   Window size    Interface control
```

The meaning of:

```
ioctl(fd, request, argument);
```

depends on:

1. The device represented by `fd`
2. The `request` code
3. The argument structure expected by that driver

## 14. A practical way to learn it

For NVMe, I recommend learning in this order:

#### Step 1 — Open the controller

```
int fd = open("/dev/nvme0", O_RDONLY);
```

#### Step 2 — Create the command

```
struct nvme_admin_cmd cmd = {0};
```

#### Step 3 — Fill the NVMe fields

```
cmd.opcode = 0x06;
cmd.nsid = 0;
cmd.addr = (uint64_t)(uintptr_t)buffer;
cmd.data_len = 4096;
cmd.cdw10 = 1;
```

#### Step 4 — Send it

```
int ret = ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);
```

#### Step 5 — Check the result

```
if (ret < 0) {
    perror("ioctl");
}
else if (ret > 0) {
    printf("NVMe status: 0x%x\n", ret);
}
else {
    printf("Success\n");
}
```

#### Step 6 — Parse the returned buffer

```
unsigned char *data = buffer;
```

### The simplest mental model

```
open()
  │
  │ "Connect me to this device"
  ▼

prepare data structure
  │
  │ "Describe what I want"
  ▼

ioctl()
  │
  │ "Ask the driver to do it"
  ▼

kernel driver
  │
  │ "Control the hardware"
  ▼

device
```

For NVMe specifically:

```
open("/dev/nvme0")
        │
        ▼
Fill struct nvme_admin_cmd
        │
        ▼
ioctl(NVME_IOCTL_ADMIN_CMD)
        │
        ▼
Linux NVMe driver
        │
        ▼
NVMe Submission Queue
        │
        ▼
NVMe Controller
        │
        ▼
Completion Queue
        │
        ▼
ioctl() returns
```

> **In short:** `ioctl()` is the bridge that lets your user-space C program request a specialized operation from a kernel device driver. For NVMe programming, it is one of the main ways to send NVMe Admin or passthrough I/O commands without writing your own kernel driver.

