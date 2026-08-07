This program sends an **NVMe Identify Controller Admin command** directly to `/dev/nvme0` using Linux `ioctl()`, receives the 4096-byte Identify Controller data structure, and prints the SSD's serial number, model number, and firmware revision.

Here is the same code explained line by line.

```c
#include <errno.h>
```

Provides definitions related to system errors, including `errno`.

In this particular program, you do not directly read `errno`, but functions such as:

```c
open()
ioctl()
```

set `errno` when they fail, and [[perror()]] uses it.

---

```c
#include <fcntl.h>
```

Provides file-control definitions, including:

```c
open()
O_RDONLY
```

You need this for:
#file_descriptor #fd
```c
open("/dev/nvme0", O_RDONLY);
```

---

```c
#include <linux/nvme_ioctl.h>
```

This is the important NVMe Linux header.

It defines structures and ioctl commands such as:

```c
struct nvme_passthru_cmd
NVME_IOCTL_ADMIN_CMD
```

Without this header, the compiler would not know what those are.

---

```c
#include <stdint.h>
```

Defines fixed-width integer types such as:

```c
uint8_t
uint16_t
uint32_t
uint64_t
uintptr_t
```

This program uses:

```c
uintptr_t
```

later when converting the buffer pointer into an integer address.

---

```c
#include <stdio.h>
```

Provides standard input/output functions such as:
#perror 
```c
printf()
fprintf()
perror()
```

---

```c
#include <stdlib.h>
```

Provides functions and constants such as:

```c
posix_memalign()
free()
EXIT_FAILURE
```

---

```c
#include <string.h>
```

Provides memory/string functions.
#memset 
This program uses:

```c
memset()
```

---

```c
#include <sys/ioctl.h>
```

Provides the declaration for:
#iotcl 
```c
ioctl()
```

The actual NVMe-specific ioctl number comes from:
#nvme_ioctl
```c
<linux/nvme_ioctl.h>
```

---

```c
#include <unistd.h>
```

Provides POSIX functions such as:
#posix
```c
close()
```

---

Now the program begins:

```c
int main(void)
```

This is the program's entry point.

`void` means the program does not expect command-line arguments.

---
#file_descriptor #fd 
```c
{
    int fd;
```

Declares an integer called `fd`.

`fd` means:

> file descriptor

Linux treats devices such as `/dev/nvme0` similarly to files.

When you open the NVMe controller:

```c
open("/dev/nvme0", ...)
```

Linux returns an integer identifying that open device.

For example:

```text
fd = 3
```

---

```c
    int status;
```

Stores the return value from:

```c
ioctl()
```

Later:

```c
status = ioctl(...);
```

The code uses this value to determine whether the command succeeded.

---

```c
    void *buffer = NULL;
```

Creates a pointer called `buffer`.

Initially:

```text
buffer
  |
  └── NULL
```

It does not point to allocated memory yet.
Later, `posix_memalign()` allocates 4096 bytes and changes `buffer` to point to that memory.

Conceptually:

```text
Before:

buffer
   |
   └── NULL


After posix_memalign():

buffer
   |
   v
+-------------------------+
|                         |
|      4096 bytes         |
|                         |
+-------------------------+
```

This buffer will receive the **Identify Controller data** from the NVMe controller.

---

```c
    struct nvme_passthru_cmd cmd;
```

Creates an NVMe passthrough command structure named:

```text
cmd
```

Linux defines this structure approximately like:

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

You fill this structure with information describing the NVMe command you want Linux to submit.

---

## Opening the NVMe controller
#fd #file_descriptor 
```c
fd = open("/dev/nvme0", O_RDONLY);
```

This opens the NVMe controller device:

```text
/dev/nvme0
```

`O_RDONLY` means:

> Open it for reading.

If successful:

```text
fd = some non-negative integer
```

For example:

```text
fd = 3
```

If unsuccessful:

```text
fd = -1
```

---

```c
if (fd < 0) {
```

Checks whether `open()` failed.
#fd #file_descriptor 
Remember:

```text
fd >= 0   success
fd < 0    failure
```

---
#perror 
```c
    perror("open");
```

Prints why `open()` failed.

For example:

```text
open: Permission denied
```

or:

```text
open: No such file or directory
```

---

```c
    return EXIT_FAILURE;
```

Stops the program and tells the operating system:

> The program failed.

`EXIT_FAILURE` is usually equivalent to a nonzero return value.

---

```c
}
```

Ends the `if` statement.

---

# Allocating the Identify buffer
#buffer 
```c
if (posix_memalign(&buffer, 4096, 4096) != 0) {
```

[[posix_memalign()]] allocates memory.

The function is:

```c
posix_memalign(
    &buffer,
    4096,
    4096
);
```

The arguments mean:

```text
&buffer   → where to store the resulting pointer

4096      → alignment

4096      → allocation size
```

So you're asking for:

> Allocate 4096 bytes whose starting address is aligned to a 4096-byte boundary.

For example:

```text
Possible aligned address:

0x100000
```

because it falls exactly on a 4096-byte boundary.

The buffer looks like:

```text
buffer
  |
  v
0x100000
+-----------------------------+
|                             |
|         4096 bytes          |
|                             |
+-----------------------------+
0x100FFF
```

The 4096-byte size is used because the **Identify Controller data structure is 4096 bytes**.

---

```c
if (... != 0)
```

`posix_memalign()` returns:

```text
0     success
nonzero failure
```

So this means:

> If allocation failed, enter this block.

---

```c
fprintf(stderr, "posix_memalign failed\n");
```

Prints an error to:

```text
stderr
```

rather than normal output.

---

```c
close(fd);
```

Closes the NVMe device because you already opened it.

This is good resource cleanup.

---

```c
return EXIT_FAILURE;
```

Stops the program.

---

# Clearing the memory

```c
memset(buffer, 0, 4096);
```

Sets all 4096 bytes of the buffer to zero.

Before:

```text
buffer:

?? ?? ?? ?? ?? ?? ?? ?? ...
```

Memory may contain arbitrary old data.

After:

```text
00 00 00 00 00 00 00 00 ...
```

This is done before the NVMe controller fills the buffer.

---

```c
memset(&cmd, 0, sizeof(cmd));
```

Clears the entire NVMe command structure.

`&cmd` means:

> address of `cmd`

`sizeof(cmd)` means:

> number of bytes occupied by the structure.

So this:

```c
memset(&cmd, 0, sizeof(cmd));
```

sets every field initially to zero:

```text
opcode       = 0
flags        = 0
nsid         = 0
addr         = 0
data_len     = 0
cdw10        = 0
cdw11        = 0
...
```

This is important because many NVMe command fields are reserved and should remain zero.

---

# Building the NVMe Identify command

```c
cmd.opcode = 0x06;
```

Sets the NVMe Admin command opcode.

NVMe opcode:

```text
0x06 = Identify
```

So this tells the controller:

> Execute an Identify command.

---

```c
cmd.nsid = 0;
```

Sets the Namespace Identifier to zero.

For:

```text
Identify Controller
```

the command is asking about the controller rather than a particular namespace, so NSID is not used to select a normal namespace here.

---

```c
cmd.addr = (uintptr_t)buffer;
```

This is very important.

`buffer` is a pointer:

```text
buffer
   |
   v
+--------------------+
| 4096-byte memory   |
+--------------------+
```

But:

```c
cmd.addr
```

is an integer field containing an address.

So:

```c
(uintptr_t)buffer
```

converts the pointer into an integer large enough to hold a memory address.

Example:

```text
buffer = 0x7F20A0100000

cmd.addr = 0x7F20A0100000
```

Linux receives this user-space address through `ioctl()`.

Important detail:

This is **not necessarily the PRP1 address that the NVMe controller eventually sees**.

Linux later maps this memory for DMA and builds the necessary PRPs/SGLs.

Conceptually:

```text
Your buffer
    |
    | virtual address
    v
cmd.addr
    |
    v
Linux NVMe driver
    |
    | DMA mapping
    v
physical/IOMMU address
    |
    v
PRP1 / PRP2
    |
    v
NVMe controller
```

---

```c
cmd.data_len = 4096;
```

Tells Linux that the command has:

```text
4096 bytes
```

of data associated with it.

For Identify Controller, the controller returns a 4096-byte Identify data structure.

---

```c
cmd.cdw10 = 0x01;
```

Sets Command DWORD 10.

For the Identify command, bits in CDW10 include:

```text
CNS
```

which means:

> Controller or Namespace Structure

Here:

```text
CNS = 0x01
```

means:

> Identify Controller.

So together:

```c
cmd.opcode = 0x06;
cmd.cdw10  = 0x01;
```

means:

> Execute Identify, requesting the Identify Controller data structure.

---

```c
cmd.timeout_ms = 5000;
```

Sets the timeout to:

```text
5000 milliseconds
```

which is:

```text
5 seconds
```

Linux should not wait indefinitely if the controller does not complete the command.

---

# Sending the NVMe command

```c
status = ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);
```

This is the key line.

You're asking Linux:

> Send this NVMe Admin command to the controller associated with `fd`.

Break it apart:

```c
ioctl(
    fd,
    NVME_IOCTL_ADMIN_CMD,
    &cmd
);
```

### `fd`

Identifies:

```text
/dev/nvme0
```

### `NVME_IOCTL_ADMIN_CMD`

Tells the Linux NVMe driver:

> This is an NVMe Admin passthrough command.

### `&cmd`

Gives Linux the address of the command structure you built.

So the complete flow is:

```text
Your program
      |
      | ioctl()
      v
Linux syscall layer
      |
      v
NVMe driver
      |
      | reads cmd
      |
      | opcode = 0x06
      | CNS = 0x01
      |
      | maps buffer
      v
NVMe Admin Submission Queue
      |
      v
SSD Controller
```

The controller executes:

```text
Identify Controller
```

and DMA-writes its 4096-byte response into your buffer.

---

# Checking whether ioctl failed

```c
if (status < 0) {
```

If `status` is negative, the Linux system call itself failed.

For example:

```text
bad file descriptor
invalid ioctl
memory mapping problem
permission problem
```

---

```c
perror("NVME_IOCTL_ADMIN_CMD");
```

Prints the corresponding Linux error.

For example:

```text
NVME_IOCTL_ADMIN_CMD: Invalid argument
```

---

# NVMe command returned an error

```c
} else if (status > 0) {
```

Here Linux successfully submitted the command, but the NVMe command completed with a nonzero status.

Conceptually:

```text
Linux ioctl worked
       |
       v
NVMe command reached SSD
       |
       v
SSD returned error status
```

---

```c
fprintf(stderr, "NVMe status: 0x%x\n", status);
```

Prints the NVMe status.

For example:

```text
NVMe status: 0x2
```

You would then decode that status according to the NVMe status fields.

---

# Command succeeded

```c
} else {
```

This means:

```text
status == 0
```

So the Identify Controller command succeeded.

At this point:

```text
buffer
```

contains the 4096-byte Identify Controller data structure.

---

```c
unsigned char *data = buffer;
```

This creates another pointer called:

```text
data
```

Both pointers point to the same memory:

```text
buffer ──────┐
             v
        +-----------+
        | Identify  |
        | Controller|
        | data      |
        +-----------+
             ^
data ────────┘
```

Why use:

```c
unsigned char *
```

Because an `unsigned char` is one byte.

So pointer arithmetic becomes very convenient:

```c
data + 4
```

means:

> Move 4 bytes into the buffer.

Not 4 integers or 4 structures—exactly 4 bytes.

---

# Serial number

```c
printf("Serial number: %.20s\n", data + 4);
```

The NVMe Identify Controller structure defines:

```text
Offset 4
Length 20 bytes
Serial Number
```

So:

```c
data + 4
```

moves the pointer 4 bytes into the Identify data structure.

Visualization:

```text
Offset

0               4
|---------------|
+-------+------------------------+
| VID   | Serial Number          |
| ...   |                        |
+-------+------------------------+
        ^
        |
      data + 4
```

The format:

```c
%.20s
```

means:

> Print at most 20 characters.

This is important because NVMe string fields aren't something you should assume are normal null-terminated C strings.

---

# Model number

```c
printf("Model number: %.40s\n", data + 24);
```

The Model Number field starts at byte offset:

```text
24
```

and is:

```text
40 bytes
```

So:

```c
data + 24
```

points to the first byte of Model Number.

And:

```c
%.40s
```

prints at most 40 characters.

---

# Firmware revision

```c
printf("Firmware revision: %.8s\n", data + 64);
```

Firmware Revision starts at:

```text
offset 64
```

and has length:

```text
8 bytes
```

Therefore:

```c
data + 64
```

points to the firmware field.

And:

```c
%.8s
```

prints exactly up to 8 characters.

---

## Why those offsets?

The beginning of the Identify Controller data looks approximately like:

```text
Byte offset

0
+-----------------------+
| VID                   | 2 bytes
+-----------------------+
2
| SSVID                 | 2 bytes
+-----------------------+
4
|                       |
| Serial Number         | 20 bytes
|                       |
+-----------------------+
24
|                       |
| Model Number          | 40 bytes
|                       |
+-----------------------+
64
| Firmware Revision     | 8 bytes
+-----------------------+
72
| ...                   |
```

Therefore:

```c
data + 4
data + 24
data + 64
```

are simply navigating to fields by their byte offsets.

---

# Releasing the buffer

```c
free(buffer);
```

Returns the 4096 bytes allocated by:

```c
posix_memalign()
```

back to the operating system.

You should pair:

```text
posix_memalign() → free()
```

when you're finished with the memory.

---

# Closing the NVMe device

```c
close(fd);
```

Closes:

```text
/dev/nvme0
```

and releases the file descriptor.

---

# Final return value

```c
return status != 0;
```

This expression evaluates to either:

```text
0
```

or:

```text
1
```

If:

```text
status == 0
```

then:

```c
status != 0
```

is false:

```text
0
```

so the program returns success.

If:

```text
status != 0
```

then the comparison is true:

```text
1
```

so the program returns failure.

Equivalent longer code would be:

```c
if (status == 0)
    return 0;
else
    return 1;
```

So:

```c
return status != 0;
```

is just a shorter version.

---

## The entire program as a flow

The easiest way to understand the program is:

```text
1. Open /dev/nvme0
          |
          v
2. Allocate 4096-byte buffer
          |
          v
3. Clear buffer and command
          |
          v
4. Build Identify Controller command

      OPC = 0x06
      CNS = 0x01
      buffer = 4096 bytes

          |
          v
5. ioctl()
          |
          v
6. Linux NVMe driver
          |
          v
7. Put command into Admin SQ
          |
          v
8. NVMe controller executes Identify
          |
          v
9. Controller DMA-writes 4096 bytes
          |
          v
10. ioctl() returns
          |
          v
11. Read:

    buffer + 4   → Serial Number
    buffer + 24  → Model Number
    buffer + 64  → Firmware Revision

          |
          v
12. free(buffer)
          |
          v
13. close(fd)
```

The three lines doing most of the actual NVMe work are therefore:

```c
cmd.opcode = 0x06;
cmd.cdw10  = 0x01;

status = ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);
```

They effectively say:

> **"Linux NVMe driver, send an Identify Controller command to `/dev/nvme0`, and place the controller's returned 4096-byte data structure into my buffer."**