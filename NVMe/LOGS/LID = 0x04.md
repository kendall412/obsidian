
# Changed Namespace List Log Page


> LID 0x04 is the Changed Namespace List log page, which reports NSIDs whose configuration or existence has changed so the host can rescan and update namespace information.

# 🔹 Retrieved Using

```
Admin Command:
  Get Log Page (Opcode 0x02)

LID:
  0x04
```

# 🔹 Purpose

Used to notify the host that namespace configuration changed, such as:

- Namespace created
- Namespace deleted
- Namespace attached/detached
- Namespace attributes modified

# 🔹 Why It Exists

In systems with:

- Hot-plug storage
- Multiple controllers
- Dynamic namespace management

the host needs a lightweight way to detect:

```
"Which namespaces changed?"
```

without rescanning everything.

# 🔹 Log Structure

The log contains an array of **Namespace Identifiers (NSIDs)**.

Structure conceptually:

```
Entry 0 → NSID
Entry 1 → NSID
Entry 2 → NSID
...
```

Each entry:

- 32-bit NSID

# 🔹 Special Value

```
NSID = FFFFFFFFh
```

means:

```
"More namespaces changed than can fit in the log"
```

👉 Host should rescan all namespaces.


# 🔹 Example

Suppose:

- Namespace 3 deleted
- Namespace 5 created

Log might return:

```
[ 0x00000003 ]
[ 0x00000005 ]
```

# 🔹 How Host Learns About Changes

Usually through:

## ✔ Asynchronous Event Request (AER)

Controller sends notice:

```
Namespace Attribute Changed
```

Then host:

1. Issues Get Log Page (LID 0x04)
2. Reads changed NSIDs
3. Re-identifies affected namespaces

# 🔹 Typical Workflow

```
1. Namespace changes internally
2. Controller raises async event
3. Host reads Changed Namespace List
4. Host updates namespace inventory
```

# 🔹 Real Usage

Common in:

- NVMe-oF
- Multi-tenant storage
- Enterprise hot-plug systems
- Dynamic namespace provisioning

# 🔹 Related Commands

|Command|Purpose|
|---|---|
|Identify Namespace|Read namespace info|
|Namespace Management|Create/delete NS|
|Namespace Attachment|Attach/detach NS|

# 🔹 Important Note

The log reports:

```
Which namespaces changed
```

NOT:

```
What changed
```

Host must re-query the namespace.


