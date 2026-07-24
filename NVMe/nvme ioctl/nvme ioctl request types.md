

> Linux provides several `ioctl` request types to interface directly with NVMe devices, primarily categorized into ==passthrough commands and controller-level management functions==. These enable user-space applications to send native NVMe specifications to both Admin and I/O submission queues. [1](https://manpages.debian.org/testing/nvme-cli/nvme.1.en.html), [2](https://github.com/torvalds/linux/blob/master/include/uapi/linux/nvme_ioctl.h)

## 1. Passthrough Commands

These are arbitrary NVMe commands that are forwarded directly to the device. [1](https://zonedstorage.io/docs/linux/overview)

- **`NVME_IOCTL_ADMIN_CMD`**: Submits Admin commands (e.g., Get Log Page, Identify) to the controller's Admin Queue.
- **`NVME_IOCTL_IO_CMD`**: Submits I/O commands to an I/O Submission Queue.
- **`NVME_IOCTL_ADMIN64_CMD`**: A 64-bit variant of the Admin command passthrough designed to handle larger data payloads and more complex command structures on modern kernels.
- **`NVME_IOCTL_IO64_CMD`**: A 64-bit variant of the I/O command passthrough. [1](https://github.com/linux-nvme/libnvme/blob/master/src/nvme/ioctl.h), [2](https://github.com/multi-stream/nvme-cli/blob/master/nvme-ioctl.c), [3](https://spdk.io/doc/nvme.html), [4](https://superuser.com/questions/1533501/getting-message-read-nvme-identify-controller-failed-nvme-ioctl-admin-cmd-ba)

## 2. Standard I/O Submissions

Instead of building a raw passthrough, the kernel provides simplified `ioctl` macros to submit standard NVM I/O operations directly.

- **`NVME_IOCTL_SUBMIT_IO`**: Used for standard block operations like `Read`, `Write`, and `Compare`. [1](https://github.com/multi-stream/nvme-cli/blob/master/nvme-ioctl.c)

## 3. Controller Management and Status

These `ioctl` commands are used to manage the state of the NVMe controller itself rather than interacting with the storage media.

- **`NVME_IOCTL_RESET`**: Performs a soft reset of the entire NVMe controller.
- **`NVME_IOCTL_SUBSYS_RESET`**: Performs a reset on the whole NVMe Subsystem (for NVMe-oF or multi-controller environments).
- **`NVME_IOCTL_ID`**: Returns the Namespace ID (NSID) associated with a specific file descriptor. [1](https://github.com/multi-stream/nvme-cli/blob/master/nvme-ioctl.c)

For detailed C-struct definitions and how the parameters are mapped into user space, you can review the [GitHub libnvme ioctl.h Source](https://github.com/linux-nvme/libnvme/blob/master/src/nvme/ioctl.h).

---
# Linux NVMe `ioctl` functions

In Linux, NVMe does not have a separate C function for every NVMe command. Applications call the generic system call:

```c
int ioctl(int fd, unsigned long request, ...);
```

The `request` argument selects an NVMe ioctl operation. The current Linux UAPI header defines **nine traditional NVMe ioctl requests** and **four asynchronous `io_uring` command requests**. ([GitHub](https://github.com/torvalds/linux/blob/master/include/uapi/linux/nvme_ioctl.h "linux/include/uapi/linux/nvme_ioctl.h at master · torvalds/linux · GitHub"))

Include these headers:

```c
#include <fcntl.h>
#include <sys/ioctl.h>
#include <linux/nvme_ioctl.h>
#include <unistd.h>
```

---

## 1. `NVME_IOCTL_ID`

```c
int nsid = ioctl(fd, NVME_IOCTL_ID);
```

Definition:

```c
#define NVME_IOCTL_ID _IO('N', 0x40)
```

Purpose:

- Returns the namespace ID associated with an NVMe namespace device.
- Normally used with a namespace file such as:
    

```text
/dev/nvme0n1
```

Example:

```c
int fd = open("/dev/nvme0n1", O_RDONLY);
if (fd < 0) {
    perror("open");
    return 1;
}

int nsid = ioctl(fd, NVME_IOCTL_ID);
if (nsid < 0) {
    perror("NVME_IOCTL_ID");
} else {
    printf("Namespace ID: %d\n", nsid);
}

close(fd);
```

The namespace ioctl handler returns the namespace’s ID directly. ([GitHub](https://github.com/torvalds/linux/blob/master/drivers/nvme/host/ioctl.c "linux/drivers/nvme/host/ioctl.c at master · torvalds/linux · GitHub"))

---

## 2. `NVME_IOCTL_ADMIN_CMD`

```c
int status = ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);
```

Definition:

```c
#define NVME_IOCTL_ADMIN_CMD \
    _IOWR('N', 0x41, struct nvme_admin_cmd)
```

`nvme_admin_cmd` is an alias:

```c
#define nvme_admin_cmd nvme_passthru_cmd
```

Purpose:

- Submit a synchronous NVMe Admin command.
    
- Examples include:
    
    - Identify
    - Get Log Page
    - Get Features
    - Set Features
    - Firmware Download
    - Firmware Commit
    - Namespace Management
    - Format NVM
    - Security Send
    - Security Receive

Typical device:

```text
/dev/nvme0
```

Command structure:

```c
struct nvme_passthru_cmd {
    __u8  opcode;
    __u8  flags;
    __u16 rsvd1;
    __u32 nsid;
    __u32 cdw2;
    __u32 cdw3;
    __u64 metadata;
    __u64 addr;
    __u32 metadata_len;
    __u32 data_len;
    __u32 cdw10;
    __u32 cdw11;
    __u32 cdw12;
    __u32 cdw13;
    __u32 cdw14;
    __u32 cdw15;
    __u32 timeout_ms;
    __u32 result;
};
```

This structure represents the important fields of the 64-byte NVMe submission queue entry. The kernel fills `result` with command-specific completion data when applicable. ([GitHub](https://github.com/torvalds/linux/blob/master/include/uapi/linux/nvme_ioctl.h "linux/include/uapi/linux/nvme_ioctl.h at master · torvalds/linux · GitHub"))

### Identify Controller example

```c
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
    int fd;
    int status;
    void *buffer = NULL;
    struct nvme_passthru_cmd cmd;

    fd = open("/dev/nvme0", O_RDONLY);
    if (fd < 0) {
        perror("open");
        return EXIT_FAILURE;
    }

    if (posix_memalign(&buffer, 4096, 4096) != 0) {
        fprintf(stderr, "posix_memalign failed\n");
        close(fd);
        return EXIT_FAILURE;
    }

    memset(buffer, 0, 4096);
    memset(&cmd, 0, sizeof(cmd));

    cmd.opcode     = 0x06;                  /* Identify */
    cmd.nsid       = 0;
    cmd.addr       = (uintptr_t)buffer;
    cmd.data_len   = 4096;
    cmd.cdw10      = 0x01;                  /* CNS = Identify Controller */
    cmd.timeout_ms = 5000;

    status = ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);

    if (status < 0) {
        perror("NVME_IOCTL_ADMIN_CMD");
    } else if (status > 0) {
        fprintf(stderr, "NVMe status: 0x%x\n", status);
    } else {
        unsigned char *data = buffer;

        printf("Serial number: %.20s\n", data + 4);
        printf("Model number: %.40s\n", data + 24);
        printf("Firmware revision: %.8s\n", data + 64);
    }

    free(buffer);
    close(fd);
    return status != 0;
}
```

## 3. `NVME_IOCTL_SUBMIT_IO`

```c
int status = ioctl(fd, NVME_IOCTL_SUBMIT_IO, &io);
```

Definition:

```c
#define NVME_IOCTL_SUBMIT_IO \
    _IOW('N', 0x42, struct nvme_user_io)
```

Purpose:

- Submit basic NVM I/O commands using `struct nvme_user_io`.
    
- Primarily intended for:
    
    - Read
        
    - Write
        
    - Compare
        

Typical device:

```text
/dev/nvme0n1
```

Structure:

```c
struct nvme_user_io {
    __u8  opcode;
    __u8  flags;
    __u16 control;
    __u16 nblocks;
    __u16 rsvd;
    __u64 metadata;
    __u64 addr;
    __u64 slba;
    __u32 dsmgmt;
    __u32 reftag;
    __u16 apptag;
    __u16 appmask;
};
```

Important detail:

```c
io.nblocks = number_of_blocks - 1;
```

For example:

```c
io.nblocks = 0;   /* one logical block */
io.nblocks = 7;   /* eight logical blocks */
```

The kernel converts this structure into an NVMe read/write-style command and submits it through the namespace queue. ([GitHub](https://github.com/torvalds/linux/blob/master/drivers/nvme/host/ioctl.c "linux/drivers/nvme/host/ioctl.c at master · torvalds/linux · GitHub"))

### Read example

```c
struct nvme_user_io io;
void *buffer = NULL;
size_t block_size = 512;

if (posix_memalign(&buffer, 4096, block_size) != 0) {
    /* Handle allocation error. */
}

memset(buffer, 0, block_size);
memset(&io, 0, sizeof(io));

io.opcode  = 0x02;                 /* NVM Read */
io.addr    = (uintptr_t)buffer;
io.slba    = 0;                    /* Starting LBA */
io.nblocks = 0;                    /* Zero-based: read one block */

int status = ioctl(fd, NVME_IOCTL_SUBMIT_IO, &io);
```

---

## 4. `NVME_IOCTL_IO_CMD`

```c
int status = ioctl(fd, NVME_IOCTL_IO_CMD, &cmd);
```

Definition:

```c
#define NVME_IOCTL_IO_CMD \
    _IOWR('N', 0x43, struct nvme_passthru_cmd)
```

Purpose:

- Submit a synchronous NVMe I/O command using the generic passthrough structure.
    
- Supports commands whose fields are expressed directly through:
    
    - `opcode`
        
    - `nsid`
        
    - `cdw10` through `cdw15`
        
    - data and metadata buffers
        

Examples:

- Read
    
- Write
    
- Compare
    
- Flush
    
- Write Zeroes
    
- Dataset Management
    
- Verify
    
- Copy
    
- Reservation commands
    
- Zone commands, where supported
    

Prefer opening the namespace device:

```text
/dev/nvme0n1
```

Using `NVME_IOCTL_IO_CMD` on the controller character device is deprecated. The current driver only permits that path when exactly one namespace exists and emits a warning. ([GitHub](https://github.com/torvalds/linux/blob/master/drivers/nvme/host/ioctl.c "linux/drivers/nvme/host/ioctl.c at master · torvalds/linux · GitHub"))

### Read example using passthrough

```c
struct nvme_passthru_cmd cmd;
void *buffer = NULL;
uint64_t slba = 0;
uint16_t block_count = 1;

posix_memalign(&buffer, 4096, 512);
memset(buffer, 0, 512);
memset(&cmd, 0, sizeof(cmd));

cmd.opcode     = 0x02;                     /* Read */
cmd.nsid       = 1;
cmd.addr       = (uintptr_t)buffer;
cmd.data_len   = 512;
cmd.cdw10      = (uint32_t)(slba & 0xffffffff);
cmd.cdw11      = (uint32_t)(slba >> 32);
cmd.cdw12      = block_count - 1;
cmd.timeout_ms = 5000;

int status = ioctl(fd, NVME_IOCTL_IO_CMD, &cmd);
```

---

## 5. `NVME_IOCTL_RESET`

```c
int status = ioctl(fd, NVME_IOCTL_RESET);
```

Definition:

```c
#define NVME_IOCTL_RESET _IO('N', 0x44)
```

Purpose:

- Reset an NVMe controller.
    
- Causes the Linux driver to run its controller reset and reinitialization procedure.
    
- Requires `CAP_SYS_ADMIN`, normally meaning root privileges.
    

Typical device:

```text
/dev/nvme0
```

Example:

```c
int fd = open("/dev/nvme0", O_RDONLY);

if (ioctl(fd, NVME_IOCTL_RESET) < 0)
    perror("NVME_IOCTL_RESET");
```

The driver checks `CAP_SYS_ADMIN` before performing the synchronous controller reset. ([GitHub](https://github.com/torvalds/linux/blob/master/drivers/nvme/host/ioctl.c "linux/drivers/nvme/host/ioctl.c at master · torvalds/linux · GitHub"))

---

## 6. `NVME_IOCTL_SUBSYS_RESET`

```c
int status = ioctl(fd, NVME_IOCTL_SUBSYS_RESET);
```

Definition:

```c
#define NVME_IOCTL_SUBSYS_RESET _IO('N', 0x45)
```

Purpose:

- Issue an NVMe subsystem reset.
    
- Potentially affects all controllers in the same NVMe subsystem.
    
- Requires `CAP_SYS_ADMIN`.
    
- The hardware must support subsystem reset.
    

Typical device:

```text
/dev/nvme0
```

Example:

```c
if (ioctl(fd, NVME_IOCTL_SUBSYS_RESET) < 0)
    perror("NVME_IOCTL_SUBSYS_RESET");
```

The Linux controller ioctl handler routes this request to the NVMe subsystem-reset implementation after checking administrative privilege. ([GitHub](https://github.com/torvalds/linux/blob/master/drivers/nvme/host/ioctl.c "linux/drivers/nvme/host/ioctl.c at master · torvalds/linux · GitHub"))

---

## 7. `NVME_IOCTL_RESCAN`

```c
int status = ioctl(fd, NVME_IOCTL_RESCAN);
```

Definition:

```c
#define NVME_IOCTL_RESCAN _IO('N', 0x46)
```

Purpose:

- Ask the driver to rescan the NVMe controller.
    
- Used to discover namespace configuration changes.
    
- Requires `CAP_SYS_ADMIN`.
    

Typical use cases:

- Namespace created
    
- Namespace deleted
    
- Namespace attached or detached
    
- Namespace attributes changed
    

Example:

```c
if (ioctl(fd, NVME_IOCTL_RESCAN) < 0)
    perror("NVME_IOCTL_RESCAN");
```

The ioctl queues an NVMe controller scan and returns without using a command structure. ([GitHub](https://github.com/torvalds/linux/blob/master/drivers/nvme/host/ioctl.c "linux/drivers/nvme/host/ioctl.c at master · torvalds/linux · GitHub"))

---

## 8. `NVME_IOCTL_ADMIN64_CMD`

```c
int status = ioctl(fd, NVME_IOCTL_ADMIN64_CMD, &cmd);
```

Definition:

```c
#define NVME_IOCTL_ADMIN64_CMD \
    _IOWR('N', 0x47, struct nvme_passthru_cmd64)
```

Purpose:

- Submit an Admin command like `NVME_IOCTL_ADMIN_CMD`.
    
- Uses a 64-bit result field.
    
- Useful for commands whose completion result needs all 64 bits.
    

Structure:

```c
struct nvme_passthru_cmd64 {
    __u8  opcode;
    __u8  flags;
    __u16 rsvd1;
    __u32 nsid;
    __u32 cdw2;
    __u32 cdw3;
    __u64 metadata;
    __u64 addr;
    __u32 metadata_len;

    union {
        __u32 data_len;
        __u32 vec_cnt;
    };

    __u32 cdw10;
    __u32 cdw11;
    __u32 cdw12;
    __u32 cdw13;
    __u32 cdw14;
    __u32 cdw15;
    __u32 timeout_ms;
    __u32 rsvd2;
    __u64 result;
};
```

Main difference:

```text
NVME_IOCTL_ADMIN_CMD    result = 32 bits
NVME_IOCTL_ADMIN64_CMD  result = 64 bits
```

([GitHub](https://github.com/torvalds/linux/blob/master/include/uapi/linux/nvme_ioctl.h "linux/include/uapi/linux/nvme_ioctl.h at master · torvalds/linux · GitHub"))

---

## 9. `NVME_IOCTL_IO64_CMD`

```c
int status = ioctl(fd, NVME_IOCTL_IO64_CMD, &cmd);
```

Definition:

```c
#define NVME_IOCTL_IO64_CMD \
    _IOWR('N', 0x48, struct nvme_passthru_cmd64)
```

Purpose:

- Submit a synchronous I/O command.
    
- Similar to `NVME_IOCTL_IO_CMD`.
    
- Uses `struct nvme_passthru_cmd64`.
    
- Returns a 64-bit command-specific result.
    

Typical device:

```text
/dev/nvme0n1
```

The namespace ioctl handler submits this request through the 64-bit passthrough processing path. ([GitHub](https://github.com/torvalds/linux/blob/master/drivers/nvme/host/ioctl.c "linux/drivers/nvme/host/ioctl.c at master · torvalds/linux · GitHub"))

---

## 10. `NVME_IOCTL_IO64_CMD_VEC`

```c
int status = ioctl(fd, NVME_IOCTL_IO64_CMD_VEC, &cmd);
```

Definition:

```c
#define NVME_IOCTL_IO64_CMD_VEC \
    _IOWR('N', 0x49, struct nvme_passthru_cmd64)
```

Purpose:

- Submit a vectored I/O passthrough command.
    
- Instead of `addr` pointing directly to one contiguous buffer, it identifies an array of I/O vectors.
    
- `vec_cnt` supplies the number of vectors.
    

Conceptually:

```c
struct iovec iov[2];

iov[0].iov_base = buffer1;
iov[0].iov_len  = length1;

iov[1].iov_base = buffer2;
iov[1].iov_len  = length2;

cmd.addr    = (uintptr_t)iov;
cmd.vec_cnt = 2;
```

The driver marks this operation with its internal vectored-I/O flag before submitting the command. ([GitHub](https://github.com/torvalds/linux/blob/master/drivers/nvme/host/ioctl.c "linux/drivers/nvme/host/ioctl.c at master · torvalds/linux · GitHub"))

---

# Summary table

|Request|Argument structure|Primary purpose|
|---|---|---|
|`NVME_IOCTL_ID`|None|Return namespace ID|
|`NVME_IOCTL_ADMIN_CMD`|`nvme_passthru_cmd`|Submit Admin command|
|`NVME_IOCTL_SUBMIT_IO`|`nvme_user_io`|Submit basic Read/Write/Compare|
|`NVME_IOCTL_IO_CMD`|`nvme_passthru_cmd`|Submit generic I/O command|
|`NVME_IOCTL_RESET`|None|Reset controller|
|`NVME_IOCTL_SUBSYS_RESET`|None|Reset NVMe subsystem|
|`NVME_IOCTL_RESCAN`|None|Rescan namespaces|
|`NVME_IOCTL_ADMIN64_CMD`|`nvme_passthru_cmd64`|Admin command with 64-bit result|
|`NVME_IOCTL_IO64_CMD`|`nvme_passthru_cmd64`|I/O command with 64-bit result|
|`NVME_IOCTL_IO64_CMD_VEC`|`nvme_passthru_cmd64`|Vectored I/O passthrough|

---

# NVMe `io_uring` command interfaces

These are defined in the same UAPI header but are not called through the traditional `ioctl()` system call:

```c
#define NVME_URING_CMD_IO         _IOWR('N', 0x80, struct nvme_uring_cmd)
#define NVME_URING_CMD_IO_VEC     _IOWR('N', 0x81, struct nvme_uring_cmd)
#define NVME_URING_CMD_ADMIN      _IOWR('N', 0x82, struct nvme_uring_cmd)
#define NVME_URING_CMD_ADMIN_VEC  _IOWR('N', 0x83, struct nvme_uring_cmd)
```

They provide asynchronous passthrough:

|Request|Purpose|
|---|---|
|`NVME_URING_CMD_IO`|Asynchronous I/O passthrough|
|`NVME_URING_CMD_IO_VEC`|Asynchronous vectored I/O passthrough|
|`NVME_URING_CMD_ADMIN`|Asynchronous Admin passthrough|
|`NVME_URING_CMD_ADMIN_VEC`|Asynchronous vectored Admin passthrough|

The current kernel requires the large `io_uring` submission and completion entry formats for NVMe passthrough: SQE128 and CQE32. ([GitHub](https://github.com/torvalds/linux/blob/master/drivers/nvme/host/ioctl.c "linux/drivers/nvme/host/ioctl.c at master · torvalds/linux · GitHub"))

# Device selection

Use the controller character device for controller-wide Admin operations:

```text
/dev/nvme0
```

Use the namespace device for NVM I/O operations:

```text
/dev/nvme0n1
```

Typical mapping:

```c
/* Controller command */
fd = open("/dev/nvme0", O_RDONLY);
ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);

/* Namespace I/O command */
fd = open("/dev/nvme0n1", O_RDWR);
ioctl(fd, NVME_IOCTL_IO_CMD, &cmd);
```

`NVME_IOCTL_ADMIN_CMD` and `NVME_IOCTL_ADMIN64_CMD` are recognized as controller operations, while namespace-specific requests are routed through the namespace ioctl handler. ([GitHub](https://github.com/torvalds/linux/blob/master/drivers/nvme/host/ioctl.c "linux/drivers/nvme/host/ioctl.c at master · torvalds/linux · GitHub"))

# Important return-value behavior

For passthrough requests:

```c
int ret = ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);
```

Interpret the result as:

```c
if (ret < 0) {
    /* Linux system or driver error; inspect errno. */
    perror("ioctl");
} else if (ret > 0) {
    /* NVMe completion status returned by the controller. */
    fprintf(stderr, "NVMe status: 0x%x\n", ret);
} else {
    /* Command completed successfully. */
}
```

Do not treat every nonzero return value as a normal Linux `errno`. A positive return can represent an NVMe command completion status rather than a system-call failure.