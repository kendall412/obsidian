#perror
`perror()` is a standard C library function that prints a **human-readable description of the most recent system/library error represented by `errno`**.

For example:

```c
#include <stdio.h>

perror("open");
```

might print:

```text
open: Permission denied
```

The string `"open"` is your label. `perror()` adds `": "` and then the error message corresponding to `errno`.

### Example with `open()`

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main(void)
{
    int fd = open("/does/not/exist", O_RDONLY);

    if (fd < 0) {
        perror("open");
        return 1;
    }

    close(fd);
    return 0;
}
```

If the file doesn't exist, `open()` fails and sets `errno`, commonly to `ENOENT`. Then:

```c
perror("open");
```

could print:

```text
open: No such file or directory
```

Conceptually:

```text
open()
  │
  ├── success → returns file descriptor
  │
  └── failure → returns -1
                    │
                    └── sets errno
                          │
                          ▼
                       perror()
                          │
                          ▼
             "No such file or directory"
```

### Why it matters in NVMe ioctl code

You might have code like:

```c
fd = open("/dev/nvme0", O_RDWR);

if (fd < 0) {
    perror("open");
    return 1;
}
```

If you don't have permission:

```text
open: Permission denied
```

Or with an NVMe ioctl:

```c
ret = ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);

if (ret < 0) {
    perror("NVME_IOCTL_ADMIN_CMD");
}
```

You might see:

```text
NVME_IOCTL_ADMIN_CMD: Invalid argument
```

This tells you the **Linux system call failed**.

One important NVMe distinction is that `perror()` is useful when `ioctl()` returns **less than 0**. If the NVMe controller itself completes the command with an NVMe error status, that's different—you need to inspect the ioctl/NVMe completion status rather than relying on `perror()`.

The basic pattern to remember is:

```c
if (something_failed) {
    perror("what failed");
}
```

`perror()` essentially means:

> **"Print why the last operation failed."**