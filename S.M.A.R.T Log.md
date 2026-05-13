
> The **NVMe SMART / Health Information Log** is a standardized log page that reports the **health, reliability, and usage statistics** of an NVMe SSD.

> **The NVMe SMART log (LID 0x02) is a 512-byte health report containing endurance, usage, temperature, and reliability metrics used to monitor SSD condition.** SMART log provides **lifetime telemetry of the SSD**, not just current status.

# 🔹 Command to Retrieve SMART Log

- **Admin Command**: Get Log Page (`Opcode 0x02`)
- **Log Identifier (LID)**: `0x02` → SMART / Health Information

Example using nvme-cli:

```
nvme smart-log /dev/nvme0
```

# 🔹 SMART Log Structure (512 bytes)

Returned as a structured data block. Below are the key fields (with byte offsets).

# 🔹 [[Critical Warning]] (Byte 0)

```
bit 0 → Available Spare below threshold
bit 1 → Temperature exceeded
bit 2 → Reliability degraded
bit 3 → Media in read-only mode
bit 4 → Volatile memory backup failed
```

👉 Any bit = 1 → warning condition

# 🔹 Composite Temperature (Bytes 1–2)

- Unit: Kelvin
- Convert to Celsius:

```
°C = K - 273
```

# 🔹 Available Spare (Byte 3)

- Remaining spare capacity (%)

# 🔹 Available Spare Threshold (Byte 4)

- Threshold below which warning triggers

# 🔹 Percentage Used (Byte 5)

```
0%   → new drive
100% → at rated endurance limit
>100 → exceeded endurance
```

# 🔹 Data Units Read (Bytes 32–47)

- 128-bit counter
- Units = **1000 × 512 bytes**

```
1 unit = 512,000 bytes
```

# 🔹 Data Units Written (Bytes 48–63)

Same unit as reads:

```
Total bytes written = value × 512,000
```

# 🔹 Host Read Commands (Bytes 64–79)

- Number of read commands issued by host

# 🔹 Host Write Commands (Bytes 80–95)

- Number of write commands issued

# 🔹 Controller Busy Time (Bytes 96–111)

- Time controller was busy
- Unit: minutes

# 🔹 Power Cycles (Bytes 112–127)

- Number of power on/off cycles

# 🔹 Power-On Hours (Bytes 128–143)

- Total hours powered on

# 🔹 Unsafe Shutdowns (Bytes 144–159)

- Power losses without proper shutdown

# 🔹 Media and Data Integrity Errors (Bytes 160–175)

- Count of unrecoverable errors

# 🔹 Error Information Log Entries (Bytes 176–191)

- Number of error log entries generated

# 🔹 Temperature Sensors (Bytes 192+)

- Up to 8 sensors
- Units: Kelvin

# 🔹 Example Interpretation

```
Percentage Used = 85%
Available Spare = 10%
Critical Warning = 0x01
```

👉 Meaning:

- Drive nearing end of life
- Spare capacity low
- Warning triggered

# 🔹 Key Metrics to Watch

## ✔ Endurance

- Percentage Used
- Data Units Written

## ✔ Reliability

- Media Errors
- Critical Warning

## ✔ Usage

- Power-on hours
- Power cycles

## ✔ Stability

- Unsafe shutdowns

# 🔹 Real-World Use

- Predict SSD failure
- Monitor wear and endurance
- Debug performance/reliability issues
- Data center fleet health monitoring

---
# Decode a sample of SMART output

> Decoding the SMART log involves converting counters (units, time, temperature) into real-world values and interpreting warning bits to assess SSD health, endurance, and reliability.

# Sample SMART Log Output

```
Smart Log for NVME device:nvme0 namespace-id:ffffffff
critical_warning                    : 0x01
temperature                         : 315 K
available_spare                     : 12%
available_spare_threshold           : 10%
percentage_used                     : 85%
data_units_read                     : 12,345,678
data_units_written                  : 23,456,789
host_read_commands                  : 345,678,901
host_write_commands                 : 456,789,012
controller_busy_time                : 12,345
power_cycles                        : 1,234
power_on_hours                      : 8,760
unsafe_shutdowns                    : 23
media_errors                        : 5
num_err_log_entries                 : 42
```

## ✔ critical_warning : 0x01

```
bit 0 → Available Spare below threshold
bit 1 → Temperature exceeded
bit 2 → Reliability degraded
bit 3 → Media in read-only mode
bit 4 → Volatile memory backup failed
```


```
0x01 → bit 0 set
```

Meaning:

- **Available spare is below threshold**

👉 This is an **active health warning**

## ✔ temperature : 315 K

Convert to Celsius:

```
315 - 273 = 42°C
```

👉 Normal operating temperature (safe range)

## ✔ available_spare : 12%

- Remaining spare NAND capacity
- Still above threshold (10%)

👉 Close to warning limit

## ✔ available_spare_threshold : 10%

- When spare drops below 10%, warning triggers

👉 Already triggered earlier (critical_warning = 1)

## ✔ percentage_used : 85%

- SSD has consumed **85% of its rated endurance**

```
Remaining life ≈ 15%
```

👉 Near end-of-life

## ✔ data_units_read : 12,345,678

Convert to bytes:

```
1 unit = 512,000 bytes

12,345,678 × 512,000 ≈ 6.32 TB read
```

## ✔ data_units_written : 23,456,789

```
23,456,789 × 512,000 ≈ 12.01 TB written
```

👉 Total host writes ≈ **12 TB**

## ✔ host_read_commands : 345,678,901

- Total read commands issued by host

## ✔ host_write_commands : 456,789,012

- Total write commands issued

👉 Useful for workload profiling (IOPS estimation)

## ✔ controller_busy_time : 12,345 minutes

Convert to hours:

```
12,345 / 60 ≈ 205.75 hours
```

👉 Time SSD was actively processing I/O

## ✔ power_cycles : 1,234

- Number of times device powered on/off

## ✔ power_on_hours : 8,760

```
8,760 hours ≈ 1 year
```

👉 Drive has been running continuously for ~1 year

## ✔ unsafe_shutdowns : 23

- Unexpected power losses

👉 Moderate risk indicator (not critical but notable)

## ✔ media_errors : 5

- Unrecoverable read/write errors

👉 Non-zero → indicates **some NAND reliability issues**

## ✔ num_err_log_entries : 42

- Total error events logged

👉 Should correlate with error log (LID 0x01)

# 🔹 Overall Health Assessment

```
Endurance     : HIGH (85% used)
Spare         : LOW (near threshold)
Warnings      : ACTIVE
Media errors  : Present
Shutdowns     : Moderate
```

👉 Conclusion:

```
Drive is nearing end-of-life and should be monitored or replaced soon
```

# 🔹 Key Insights

- **Percentage Used + Spare** → best indicators of wear
- **Data Units Written** → workload magnitude
- **Media Errors** → reliability red flag
- **Critical Warning** → immediate attention


---

# If you want, I can:

- decode a **real SMART log output line-by-line**
- or compute **DWPD/endurance from SMART counters**

