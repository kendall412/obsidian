
Let's walk through a realistic example of how **SPDK** is used in an NVMe validation test to verify a **Read** command. This is similar to what an NVMe validation engineer might write, although production code is usually more modular and includes extensive error handling and logging.

---

# Validation Goal

Suppose the test case is:

> Verify that the SSD correctly returns data written to Logical Block Address (LBA) 100.

The validation procedure is:

```text
1. Initialize SPDK
2. Discover the NVMe controller
3. Attach to the controller
4. Open Namespace 1
5. Allocate a DMA buffer
6. Write a known pattern to LBA 100
7. Read LBA 100
8. Compare the returned data with the original pattern
9. Report PASS or FAIL
```

---

# Overall Architecture

```text
Validation Test
      │
      ▼
SPDK Application
      │
      ▼
SPDK NVMe Library
      │
      ▼
PCIe MMIO Registers
      │
      ▼
NVMe Controller
      │
      ▼
NAND Flash
```

---

# Step 1 — Initialize SPDK

The application first initializes the SPDK environment.

```c
spdk_env_opts opts;

spdk_env_opts_init(&opts);

opts.name = "nvme_validation";

spdk_env_init(&opts);
```

This prepares hugepage memory, DMA support, and PCI access.

---

# Step 2 — Discover Controllers

SPDK scans the PCI bus.

```c
spdk_nvme_probe(
    NULL,
    NULL,
    probe_cb,
    attach_cb,
    NULL);
```

Internally:

```text
Scan PCI Bus

↓

Find NVMe Device

↓

Read PCI Configuration Space

↓

Map BAR0

↓

Initialize Controller

↓

Identify Controller
```

The `attach_cb()` callback is invoked for each controller found.

---

# Step 3 — Get Namespace

Once attached:

```c
struct spdk_nvme_ns *ns;

ns = spdk_nvme_ctrlr_get_ns(ctrlr, 1);
```

Now the program has access to Namespace 1.

---

# Step 4 — Allocate DMA Buffer

Unlike ordinary memory allocated with `malloc()`, NVMe DMA buffers must meet alignment and accessibility requirements.

```c
void *buffer;

buffer = spdk_dma_zmalloc(
            4096,
            4096,
            NULL);
```

The result is a DMA-safe 4 KB buffer.

---

# Step 5 — Fill Test Pattern

For example:

```c
memset(buffer, 0x5A, 4096);
```

Now the buffer contains:

```text
5A 5A 5A 5A
5A 5A 5A 5A
5A 5A 5A 5A
...
```

---

# Step 6 — Write Data

Submit a write command.

```c
spdk_nvme_ns_cmd_write(
    ns,
    qpair,
    buffer,
    100,
    8,
    write_complete,
    NULL,
    0);
```

Meaning:

|Parameter|Value|
|---|---|
|Namespace|ns|
|Queue Pair|qpair|
|Buffer|DMA buffer|
|Starting LBA|100|
|Number of Blocks|8|
|Callback|write_complete|

Internally SPDK performs:

```text
Build NVMe Write Command

↓

Fill PRP List

↓

Write Submission Queue Entry

↓

Ring Submission Doorbell

↓

SSD Executes Write

↓

Completion Queue Updated
```

---

# Step 7 — Wait for Completion

SPDK uses polling.

```c
while (!done)
{
    spdk_nvme_qpair_process_completions(
        qpair,
        0);
}
```

The completion callback is called when the SSD finishes the write.

---

# Step 8 — Clear Buffer

To ensure the subsequent read actually returns data from the SSD:

```c
memset(buffer, 0, 4096);
```

Now the buffer is:

```text
00 00 00 00
00 00 00 00
...
```

---

# Step 9 — Read Data

Issue the read command.

```c
spdk_nvme_ns_cmd_read(
    ns,
    qpair,
    buffer,
    100,
    8,
    read_complete,
    NULL,
    0);
```

SPDK builds:

```text
Opcode = READ

↓

LBA = 100

↓

NLB = 8

↓

PRP1 = DMA Buffer

↓

Submit SQ Entry

↓

Ring Doorbell
```

The SSD performs:

```text
Read NAND

↓

ECC Correction

↓

FTL Address Translation

↓

DMA Data to Host Buffer

↓

Write Completion Queue Entry
```

---

# Step 10 — Process Completion

Again:

```c
while (!done)
{
    spdk_nvme_qpair_process_completions(
        qpair,
        0);
}
```

Once the callback runs, the buffer contains the data transferred by the SSD.

---

# Step 11 — Validate Data

Compare the returned data with the expected pattern.

```c
for (int i = 0; i < 4096; i++)
{
    if (((uint8_t *)buffer)[i] != 0x5A)
    {
        printf("FAIL\n");
        return;
    }
}

printf("PASS\n");
```

In production code, engineers would typically use `memcmp()`:

```c
if (memcmp(buffer, expected_buffer, 4096) == 0)
{
    PASS;
}
else
{
    FAIL;
}
```

---

# What Happens Inside the SSD

```text
Host
 │
 │ READ LBA=100
 ▼
Submission Queue
 │
 ▼
Controller Fetches Command
 │
 ▼
FTL Maps LBA→Physical NAND Page
 │
 ▼
Read NAND
 │
 ▼
ECC Correction
 │
 ▼
DMA Data to Host Memory
 │
 ▼
Completion Queue Entry Written
 │
 ▼
Host Polls CQ
 │
 ▼
Validation Compares Data
```

---

# How a Real Validation Framework Wraps This

Validation frameworks usually hide the low-level SPDK calls behind helper functions. Instead of calling `spdk_nvme_ns_cmd_read()` directly in every test, a framework might provide:

```cpp
ssd.write(lba, pattern);
ssd.read(lba, read_buffer);

EXPECT_EQ(pattern, read_buffer);
```

Internally, those wrapper functions handle:

- DMA buffer allocation
    
- Queue pair management
    
- Completion polling
    
- Timeouts
    
- Logging
    
- Error handling
    
- Data comparison
    

This makes individual test cases much shorter and easier to maintain.

## Example of a complete test case

A simplified validation test might look like this:

```text
TEST(ReadBasic)
{
    pattern = GeneratePattern(0x5A);

    SSD.Write(LBA=100, pattern);

    data = SSD.Read(LBA=100);

    ASSERT(data == pattern);
}
```

Although this appears simple, each `Write()` and `Read()` call may execute hundreds of lines of SPDK-based code behind the scenes to build commands, manage DMA memory, process completions, handle errors, and record detailed logs. This abstraction is one reason large NVMe validation frameworks can contain thousands of readable test cases while sharing a common SPDK infrastructure underneath.