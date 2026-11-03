
If your goal is to develop NVMe SSD validation scripts using SPDK, I recommend starting with a small C-based test application that can:
1. Discover an NVMe controller.
2. Initialize the controller.
3. Identify the controller and namespace.
4. Create an I/O queue pair.
5. Submit NVMe Read and Write commands.
6. Retrieve and validate completion status.
7. Expand into Admin commands, negative testing, and FDP validation.


SPDK (Storage Performance Development Kit) is particularly useful for NVMe validation because it allows user-space applications to communicate directly with NVMe controllers without relying on the Linux kernel's normal NVMe block I/O path.

## 1. Understand how SPDK accesses NVMe

![[understanding spdk access.png]]

SPDK gives you programmatic access to NVMe controller operations, including command submission, completion processing, queue management, and data buffers.

One important distinction: SPDK does not eliminate the PCIe or NVMe protocols. It replaces much of the host software path used to communicate with the device.

## 2. Install SPDK on Linux

I recommend Ubuntu on a dedicated SSD validation host, with at least one non-boot NVMe test drive.

### Step 1: Download SPDK

```
git clone https://github.com/spdk/spdk --recursive
cd spdk
```

### Step 2: Install dependencies and compile

```
sudo scripts/pkgdep.sh

./configure
make -j$(nproc)
```

### Step 3: Identify the SSD's PCIe address

```
lspci -D | grep -i "non-volatile"
```

Example:

```
0000:03:00.0 Non-Volatile memory controller: NVMe SSD
```

### Step 4: Prepare hugepages and device access

SPDK requires DMA-capable memory and access to the PCIe controller.

Warning: `scripts/setup.sh` can unbind NVMe devices from the Linux kernel driver, potentially including devices holding mounted filesystems. Use a dedicated test host, verify the target BDF, and avoid unbinding the boot drive.

```
sudo HUGEMEM=2048 scripts/setup.sh
```

Inspect the device binding:

```
sudo scripts/setup.sh status
```

The official SPDK guide describes the build and device preparation process.

### Step 5: Run the first NVMe example

Start with a read-only Identify test:

```
sudo build/bin/spdk_nvme_identify
```

Then examine the NVMe Hello World example:

```
examples/nvme/hello_world/hello_world.c
```

The Hello World example demonstrates controller discovery, namespace access, queue allocation, I/O submission, and completion processing.

## 3. Learn these SPDK NVMe APIs first

|API|Purpose|
|---|---|
|`spdk_env_init()`|Initialize the SPDK runtime|
|`spdk_nvme_probe()`|Discover and attach NVMe controllers|
|`spdk_nvme_ctrlr_get_data()`|Access Identify Controller data|
|`spdk_nvme_ctrlr_get_ns()`|Obtain a namespace object|
|`spdk_nvme_ns_get_sector_size()`|Get namespace logical block size|
|`spdk_nvme_ctrlr_alloc_io_qpair()`|Create an I/O queue pair|
|`spdk_dma_zmalloc()`|Allocate DMA-capable data buffers|
|`spdk_nvme_ns_cmd_read()`|Submit a Read command|
|`spdk_nvme_ns_cmd_write()`|Submit a Write command|
|`spdk_nvme_qpair_process_completions()`|Poll for I/O completions|
|`spdk_nvme_ctrlr_cmd_admin_raw()`|Submit a raw Admin command|
|`spdk_nvme_ctrlr_cmd_io_raw()`|Submit a raw I/O command|

These APIs are documented in the [SPDK NVMe API reference](https://spdk.io/doc/nvme_8h.html?utm_source=chatgpt.com).


## 4. Understand the NVMe test execution flow

1. Initialize SPDK - Initialize environment and hugepage memory
2. Discover controller - Find the SSD using its PCIe BDF
3. Get namespace - Retrieve NSID and logical block size
4. Allocate I/O qpair - Establish Submission and Completion Queues
5. Allocate DMA buffer - Prepare the data transfer memory
6. Submit NVMe command - Write an SQE through the SPDK driver
7. Poll completions - Process CQEs and inspect status
8. Validate results - Compare actual and expected outcomes

Unlike a synchronous `ioctl()` call, SPDK commonly uses asynchronous command submission and completion polling.

The submission function returning zero generally means the command was successfully queued, not that the SSD completed it successfully.

You must inspect the NVMe completion.

## 5. Your first C test: NVMe Identify Controller

A good first test is to retrieve Identify Controller data and validate that the command succeeds.

This example uses SPDK's raw Admin command interface.

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdint.h>
#include <string.h>
#include <spdk/env.h>
#include <spdk/nvme.h>
#include <spdk/nvme_spec.h>

static struct spdk_nvme_ctrlr *g_ctrlr = NULL;

struct test_result {
    bool done;
    bool success;
    uint16_t status;
};

static bool probe_cb(void *ctx,
                     const struct spdk_nvme_transport_id *trid,
                     struct spdk_nvme_ctrlr_opts *opts)
{
    return true;
}

static void attach_cb(void *ctx,
                      const struct spdk_nvme_transport_id *trid,
                      struct spdk_nvme_ctrlr *ctrlr,
                      const struct spdk_nvme_ctrlr_opts *opts)
{
    if (g_ctrlr == NULL) {
        g_ctrlr = ctrlr;
    }
}

static void completion_cb(void *arg,
                          const struct spdk_nvme_cpl *cpl)
{
    struct test_result *result = arg;

    result->success = !spdk_nvme_cpl_is_error(cpl);
    result->status = cpl->status.sc;
    result->done = true;
}

int main(void)
{
    struct spdk_env_opts opts;
    struct spdk_nvme_cmd cmd = {0};
    struct test_result result = {0};
    void *buffer = NULL;
    int rc = 1;

    spdk_env_opts_init(&opts);
    opts.name = "nvme_identify_test";

    if (spdk_env_init(&opts) < 0) {
        fprintf(stderr, "SPDK initialization failed\n");
        return 1;
    }

    if (spdk_nvme_probe(NULL, NULL, probe_cb,
                        attach_cb, NULL) != 0 || !g_ctrlr) {
        fprintf(stderr, "No NVMe controller found\n");
        goto cleanup;
    }

    buffer = spdk_dma_zmalloc(4096, 4096, NULL);
    if (!buffer) {
        fprintf(stderr, "DMA allocation failed\n");
        goto cleanup;
    }

    cmd.opc = 0x06;   /* Identify */
    cmd.nsid = 0;
    cmd.cdw10 = 0x01; /* CNS: Identify Controller */

    if (spdk_nvme_ctrlr_cmd_admin_raw(
            g_ctrlr, &cmd, buffer, 4096,
            completion_cb, &result) != 0) {
        fprintf(stderr, "Command submission failed\n");
        goto cleanup;
    }

    while (!result.done) {
        int n = spdk_nvme_ctrlr_process_admin_completions(
            g_ctrlr);

        if (n < 0) {
            fprintf(stderr, "Completion polling failed\n");
            goto cleanup;
        }
    }

    if (!result.success) {
        printf("FAIL: NVMe status code = 0x%x\n",
               result.status);
        goto cleanup;
    }

    printf("PASS: Identify Controller completed\n");

    /* Identify Controller serial number: bytes 4-23 */
    printf("Serial Number: %.20s\n",
           (char *)buffer + 4);

    rc = 0;

cleanup:
    spdk_dma_free(buffer);

    if (g_ctrlr) {
        spdk_nvme_detach(g_ctrlr);
    }

    spdk_env_fini();
    return rc;
}
```

This is a learning example, not a production-ready test harness. It probes all accessible controllers and uses the first attached one. On a validation system, you should filter by PCIe BDF and add completion timeouts, transport error handling, and full SCT/SC decoding.

For compilation, use SPDK's existing example Makefiles or build integration rather than manually guessing library dependencies.

## 6. How to develop more advanced NVMe tests

Once Identify Controller works, I recommend this progression:

|Stage|Test|SPDK capability|
|---|---|---|
|1|Identify Controller|Admin command|
|2|Identify Namespace|Admin command|
|3|Read LBA|I/O command|
|4|Write and read-back verification|I/O and data comparison|
|5|Dataset Management Deallocate|I/O command|
|6|Get Log Page|Admin command|
|7|Set/Get Features|Admin commands|
|8|Invalid opcode and field testing|Raw command interface|
|9|Queue depth and concurrent I/O|Multiple outstanding commands|
|10|FDP configuration, logs, and RUH Update|Raw Admin/I/O management commands|

For destructive tests, use a dedicated test namespace and an explicit test-range guard.

## 7. Using SPDK for FDP validation

SPDK is useful for FDP because you can construct raw commands when a high-level API is unavailable.

For example:

```c
struct spdk_nvme_cmd cmd = {0};

cmd.opc = 0x02;     /* Get Log Page */
cmd.nsid = 0;       /* Command-specific scope */

/* Populate CDW10-CDW14 for the FDP log request. */
```

Then submit using:
```c
spdk_nvme_ctrlr_cmd_admin_raw(
    ctrlr,
    &cmd,
    buffer,
    buffer_size,
    completion_cb,
    &result
);
```

For FDP validation, this approach allows you to control the log identifier, log-specific identifier, transfer length, and other command fields. You can similarly use raw I/O commands for I/O Management Send/Receive and placement-related testing.

One caveat: SPDK manages transport details such as command identifiers and data pointers. Raw commands let you control many NVMe fields, but they do not necessarily let you inject arbitrary malformed PCIe transactions or bypass all driver validation.

## 8. Recommended architecture: Python test scripts + SPDK C backend

For an expandable NVMe validation framework, I recommend separating test orchestration from low-level NVMe command execution.

```text
nvme_validation/
│
├── tests/
│   ├── test_identify.py
│   ├── test_read_write.py
│   ├── test_deallocate.py
│   └── test_fdp.py
│
├── framework/
│   ├── test_runner.py
│   ├── result_validator.py
│   └── device_manager.py
│
├── spdk_backend/
│   ├── nvme_controller.c
│   ├── nvme_admin.c
│   ├── nvme_io.c
│   └── nvme_fdp.c
│
└── main.py
```

Python can manage test cases, parameters, expected statuses, logging, and reports. The C backend uses SPDK to communicate with the SSD.

You can expose the backend through a shared library, Python extension, or controlled subprocess interface. SPDK also provides Python tooling and bindings, but not every low-level NVMe API is necessarily exposed as a convenient Python method.

## 9. Best resources to study

- [SPDK Getting Started](https://spdk.io/doc/getting_started.html?utm_source=chatgpt.com) — installation, building, and environment setup.
- [SPDK NVMe Driver Guide](https://spdk.io/doc/nvme.html?utm_source=chatgpt.com) — controller discovery, I/O queues, and driver architecture.
- [NVMe Hello World source](https://github.com/spdk/spdk/blob/master/examples/nvme/hello_world/hello_world.c?utm_source=chatgpt.com) — a practical first C application.
- [NVMe API Reference](https://spdk.io/doc/nvme_8h.html?utm_source=chatgpt.com) — detailed function signatures and parameters.
    
My recommendation: Start by compiling and modifying `examples/nvme/hello_world/hello_world.c`. Make it perform Identify Controller, then add one Read command and CQE validation. Once you understand that execution flow, you can build a reusable command interface for FDP, namespace management, error injection, and other NVMe specification validation tests.

The most important initial choice is whether you want to write NVMe tests primarily in Python, with SPDK underneath, or develop the tests directly in C using SPDK. That determines how you should structure your test framework.