
In the PCIe Base Specification, a **Completer** is the PCIe Function or component that **receives a Request TLP addressed to it, performs the requested operation, and—when required—returns a Completion TLP to the Requester**.

The easiest way to think about it is:
```
Requester                         Completer
    │                                 │
    │       Request TLP               │
    │────────────────────────────────►│
    │                                 │
    │       Completion TLP            │
    │◄────────────────────────────────│
    │                                 │
```

So:

> **Requester = initiates a transaction**  
> **Completer = services the transaction**

#### Example: Configuration Read
From your previous question, suppose the Root Complex wants to read the Vendor ID of an NVMe SSD.

```
Root Complex                         NVMe SSD
(Requester)                         (Completer)
     │                                  │
     │    Configuration Read Request    │
     │─────────────────────────────────►│
     │                                  │
     │                                  │ Read
     │                                  │ Config Space
     │                                  ▼
     │                                  │
     │    Completion with Data (CplD)   │
     │◄─────────────────────────────────│
```

In this transaction:
```
Requester = Root Complex
Completer = NVMe Endpoint
```

The NVMe Endpoint is called the **Completer** because the Configuration Request targets it.

#### Example: Memory Read
The relationship can reverse.

Suppose an NVMe SSD needs to DMA-read data from host memory.
The NVMe controller generates a Memory Read Request:
```
NVMe SSD                           Root Complex /
Requester                          Host Memory
                                   Completer
   │                                   │
   │       Memory Read Request         │
   │──────────────────────────────────►│
   │                                   │
   │                                   │ Read memory
   │                                   ▼
   │                                   │
   │      Completion with Data         │
   │◄──────────────────────────────────│
```

Now:
```
Requester = NVMe SSD
Completer = host side
```

So **"Endpoint" and "Completer" are not synonyms**. A device can act as a Requester in one transaction and a Completer in another.

### Requests that require Completions
An important PCIe distinction is that **not every Request generates a Completion**.

For example: #post #non-post

|Request|Completion expected?|
|---|---|
|Memory Read|**Yes**|
|Memory Write|**No**|
|Configuration Read|**Yes**|
|Configuration Write|**Yes**|
|I/O Read|**Yes**|
|I/O Write|**Yes**|

A Memory Write is a **Posted Request**. The Requester sends it without waiting for a Completion:
```
Requester                    Target
    │                           │
    │     Memory Write         │
    │─────────────────────────►│
    │                           │
    │      No Completion       │
    │         ◄──── X          │
```

A Memory Read is **Non-Posted**, so a Completion is required:
```
Requester                    Completer
    │                           │
    │      Memory Read         │
    │─────────────────────────►│
    │                           │
    │       CplD               │
    │◄─────────────────────────│
```

### What does the Completer return?
PCIe defines Completion TLPs including:
```
Cpl   = Completion without Data
CplD  = Completion with Data
```

For a successful read:
```
Memory Read
     │
     ▼
Completer
     │
     └────► CplD
            +
            requested data
```

For a Configuration Write, for example:
```
Configuration Write
       │
       ▼
   Completer
       │
       └────► Cpl
```

### The Completion identifies the original Request

PCIe may have many transactions outstanding simultaneously:

```
Requester

Request A ───────►
Request B ───────►
Request C ───────►
Request D ───────►
```

Therefore, when a Completion comes back, the Requester must know which Request it belongs to.

Important fields include the **Requester ID** and **Tag**.

Conceptually:

```
Original Request

Requester ID = 03:00.0
Tag          = 0x25
Address      = ...
```

The Completion carries information that allows it to be associated with that outstanding request:

```
Completion

Requester ID = 03:00.0
Tag          = 0x25
```

So the Requester can determine:

```
CplD Tag 0x25
       │
       ▼
matches
       │
       ▼
Memory Read Request Tag 0x25
```

This is especially important because PCIe can have **many outstanding reads**.

### Completer ID

A Completion also contains a **Completer ID**, identifying the Function that generated the Completion.

Conceptually:

```
Completion TLP

┌─────────────────────────────┐
│ Completion header           │
├─────────────────────────────┤
│ Completer ID                │
├─────────────────────────────┤
│ Completion Status           │
├─────────────────────────────┤
│ Byte Count                  │
├─────────────────────────────┤
│ Requester ID                │
├─────────────────────────────┤
│ Tag                         │
├─────────────────────────────┤
│ Lower Address               │
├─────────────────────────────┤
│ Data (CplD only)            │
└─────────────────────────────┘
```

Notice that both identities matter:

```
Requester ID
    │
    └── Who originally requested it?

Completer ID
    │
    └── Who completed it?
```

### Completion status

The Completer also reports whether it successfully serviced the Request.

Examples include:

```
SC  = Successful Completion
UR  = Unsupported Request
CA  = Completer Abort
CRS = Configuration Request Retry Status
```

For example:

```
Requester                         Completer
    │                                 │
    │ Configuration Read             │
    │───────────────────────────────►│
    │                                 │
    │         CplD                    │
    │         Status = SC             │
    │◄───────────────────────────────│
```

Or if the Request isn't supported:

```
Requester                         Completer
    │                                 │
    │ Request                         │
    │───────────────────────────────►│
    │                                 │
    │ Cpl                             │
    │ Status = UR                     │
    │◄───────────────────────────────│
```

### Putting Requester and Completer together

For an NVMe SSD, you will commonly see both directions:

```
1. Host reads NVMe/PCIe device information

Root Complex ─── Request ───► NVMe SSD
 Requester                    Completer


2. NVMe controller DMA-reads host memory

NVMe SSD ─────── Request ───► Host
Requester                     Completer


3. NVMe controller DMA-writes host memory

NVMe SSD ─── Memory Write ──► Host

Posted Request:
no Completion is returned
```

So when reading the PCIe Base Specification, don't think of **Completer** as a particular physical device.

Think of it as a **role in a transaction**:

> **The Requester originates the Request. The Completer is the component responsible for servicing that Request and returning a Completion when the transaction requires one.**

This Requester/Completer model is fundamental for understanding **TLP headers, Tags, Requester ID, Completer ID, Completion Status, and PCIe transaction ordering**.