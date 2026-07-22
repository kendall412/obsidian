[spdk home page](https://spdk.io/)

> **SPDK (Storage Performance Development Kit)** is an open-source user-space software framework for developing high-performance storage applications. It provides libraries that allow software to communicate directly with NVMe SSDs from user space, bypassing the Linux kernel NVMe driver for most I/O operations.

For NVMe validation automation, SPDK is often used because it gives engineers **fine-grained control over NVMe commands**, **lower latency**, and **higher performance** than traditional kernel interfaces.

---

# Why SPDK was created

Normally, an application accesses an NVMe SSD like this:

```text
Application
      │
      ▼
Linux System Call
      │
      ▼
Linux NVMe Driver
      │
      ▼
PCIe
      │
      ▼
NVMe SSD
```

Every I/O request crosses the kernel boundary, which adds overhead due to:

- System calls
    
- Context switches
    
- Interrupt handling
    
- Kernel scheduling
    

This is perfectly acceptable for most applications, but it can limit maximum performance and make certain types of validation more difficult.

---

# SPDK architecture

SPDK moves the NVMe driver into user space.

```text
          Validation Program
                 │
                 ▼
           SPDK NVMe Library
                 │
          User-space PCI Driver
                 │
                 ▼
             PCIe Hardware
                 │
                 ▼
              NVMe SSD
```

Instead of using the kernel NVMe driver, SPDK typically uses the **VFIO** (or previously UIO) framework to map the PCIe device directly into user space.

---

# Traditional ioctl() vs SPDK

## Traditional Linux approach

```text
Python Test
      │
      ▼
ioctl()
      │
      ▼
Linux NVMe Driver
      │
      ▼
SSD
```

Advantages:

- Simple to use
    
- Uses standard Linux interfaces
    
- Good for functional testing
    
- Supports all normal operating system features
    

Disadvantages:

- Higher overhead
    
- Kernel scheduling affects timing
    
- Less control over queue processing
    

---

## SPDK approach

```text
Python/C Program
       │
       ▼
SPDK Library
       │
       ▼
Direct PCIe Access
       │
       ▼
NVMe SSD
```

Advantages:

- Very low latency
    
- Millions of IOPS possible
    
- Complete control over submission/completion queues
    
- Polling instead of interrupts
    
- Excellent for performance and stress testing
    

---

# How SPDK works internally

Suppose a validation test wants to read one block.

The application calls:

```c
spdk_nvme_ns_cmd_read(...)
```

Internally SPDK performs:

```text
Build NVMe Command
        │
        ▼
Allocate DMA Buffer
        │
        ▼
Fill PRP Entries
        │
        ▼
Write Submission Queue Entry
        │
        ▼
Ring Doorbell
        │
        ▼
Poll Completion Queue
        │
        ▼
Return Data
```

The application doesn't need to manually program all of these steps.

---

# SPDK uses polling

The Linux driver normally uses interrupts.

```text
Command Sent
      │
      ▼
SSD finishes
      │
      ▼
Interrupt
      │
      ▼
CPU wakes
```

SPDK instead continuously checks the completion queue.

```text
Command Sent
      │
      ▼
CPU Polls CQ
      │
      ▼
Completion Appears
      │
      ▼
Continue Immediately
```

Polling eliminates interrupt latency, which is one reason SPDK achieves very high performance.

---

# Example validation program

A simplified C example looks like:

```c
connect_to_controller();

identify_controller();

read_block();

write_block();

flush();

disconnect();
```

Each of these functions uses SPDK APIs underneath.

---

# Typical validation workflow

A validation program might:

```text
Initialize SPDK
        │
        ▼
Discover Controllers
        │
        ▼
Identify Controller
        │
        ▼
Create Queue Pair
        │
        ▼
Issue Commands
        │
        ▼
Verify Results
        │
        ▼
Destroy Queue Pair
```

---

# Example automation

Suppose a test wants to verify 1,000 random reads.

Without SPDK:

```text
for 1000 reads

↓

Kernel Driver

↓

Interrupt

↓

Completion
```

With SPDK:

```text
for 1000 reads

↓

SPDK Queue Pair

↓

Poll CQ

↓

Completion
```

This generally completes faster because there is less software overhead.

---

# What validation engineers use SPDK for

SPDK is especially useful for:

- High-IOPS performance testing
    
- Latency measurements
    
- Queue-depth testing
    
- Queue pair validation
    
- Long-duration stress testing
    
- Multi-threaded workload generation
    
- Custom NVMe command development
    
- Vendor-specific admin commands
    

For example, a vendor-specific Admin command can be built directly with SPDK structures and submitted without waiting for kernel support.

---

# SPDK vs nvme-cli

|Feature|SPDK|`nvme-cli`|
|---|---|---|
|Primary use|Library/framework for custom applications|Command-line utility|
|Programming interface|C API (with language bindings available)|Shell commands|
|Performance|Very high|Adequate for administration and many tests|
|Kernel dependency|Bypasses kernel data path|Uses Linux NVMe driver|
|Queue control|Direct control|Limited|
|Best for|Automated performance/stress tools|Functional testing, scripting, diagnostics|

---

# Is SPDK used everywhere?

No. Validation teams typically use multiple interfaces depending on the goal:

- **`nvme-cli`**: Administrative commands, functional validation, quick scripting, compliance checks.
    
- **`ioctl()`**: Custom Linux-based test tools that still use the kernel driver.
    
- **SPDK**: High-performance workloads, queue behavior, latency, and stress testing.
    
- **Vendor-specific APIs/tools**: Manufacturing tests, firmware download, debug features, and internal diagnostics.
    

---

# Should you learn SPDK?

Given your recent questions about NVMe firmware, `ioctl()`, and validation automation, learning SPDK is a logical next step if you're targeting firmware or NVMe validation engineering.

A practical learning path is:

1. Master the NVMe command set and queue architecture.
    
2. Learn Linux NVMe programming with `ioctl()`.
    
3. Learn the SPDK programming model (controllers, namespaces, queue pairs, DMA buffers).
    
4. Build simple SPDK programs such as:
    
    - Identify Controller
        
    - Read/Write a block
        
    - Get SMART/Health Log
        
    - Run sequential and random I/O
        
5. Use SPDK to develop automated performance and stress tests.
    

Understanding both the kernel-based (`ioctl()`) and SPDK approaches is valuable because many validation environments use both, choosing the interface that best fits the type of testing being performed.