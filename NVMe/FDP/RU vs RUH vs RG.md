
In NVMe Flexible Data Placement (FDP), Reclaim Groups (RGs), Reclaim Units (RUs), and Reclaim Unit Handles (RUHs) work together to determine where the SSD controller places data and how that data is subsequently reclaimed.

The simplest way to understand their relationship is:

- Reclaim Group (RG): A collection of Reclaim Units that share a common reclamation domain.
    
- Reclaim Unit (RU): A unit of physical media capacity that the controller manages for data placement and reclamation.
    
- Reclaim Unit Handle (RUH): A logical handle used to direct writes to a particular Reclaim Unit within a Reclaim Group.
    

The key distinction is that an RUH is not itself a Reclaim Unit. It is an abstraction that the controller uses to associate incoming writes with an RU.

## 1. Relationship between RG, RUH, and RU

Each Reclaim Group contains multiple Reclaim Units. RUHs provide logical associations to RUs within their respective groups. The illustration simplifies the actual mapping.

An important detail: the same RUH index can participate in placement across multiple Reclaim Groups. The PID identifies both the Reclaim Group and the Placement Handle, which determines the selected RUH.

## 2. Detailed explanation of each component

### Reclaim Group (RG)

A Reclaim Group contains one or more Reclaim Units.

The controller organizes its FDP storage into Reclaim Groups. A group represents a reclamation domain, allowing data placement to be separated across different groups.

For example, a controller might implement:

```
Endurance Group
│
├── Reclaim Group 0
│   ├── RU 0
│   ├── RU 1
│   ├── RU 2
│   └── RU 3
│
└── Reclaim Group 1
    ├── RU 0
    ├── RU 1
    ├── RU 2
    └── RU 3
```

An RG may correspond to a set of NAND dies or another firmware-defined physical organization. The specification does not require one RG to equal one NAND die.

![](https://www.google.com/s2/favicons?domain=https://semiconductor.samsung.com&sz=32)

Samsung Semiconductor Global

### Reclaim Unit (RU)

A Reclaim Unit represents a unit of nonvolatile storage that can be erased independently of other Reclaim Units.

An RU typically contains multiple NAND pages or erase blocks, depending on the SSD's physical architecture.

An RU has a finite amount of writable capacity. When it becomes full, the controller can associate its RUH with another available RU.

For example:

```
RU-A (capacity 1000 media-write units)

Before writes:
[____________________] 0% written

After writes:
[##########__________] 50% written

After more writes:
[####################] 100% written
```

The data in RU-A remains valid until it is invalidated or relocated and the controller can reclaim the RU.

### Reclaim Unit Handle (RUH)

An RUH is a controller-managed logical resource that references one RU in each Reclaim Group.

This is an important distinction: an RUH is not limited to a single RG. The same RUH identifier can reference different RUs across multiple RGs.

For example:

|RUH|Reclaim Group 0|Reclaim Group 1|
|---|---|---|
|RUH 0|RU-A|RU-D|
|RUH 1|RU-B|RU-E|
|RUH 2|RU-C|RU-F|

This means RUH 1 references RU-B in RG 0 and RU-E in RG 1.

The controller can change these references as RUs fill or are reassigned. A particular RU can be referenced by at most one RUH at a time.

## 3. How the host selects an RU using a PID

The host does not normally specify an internal RU identifier directly.

Instead, it uses a Placement Identifier (PID) in a supported write command.

The selection path is:

![[selection path.png]]

### Example

Assume:
- NRG = 2
- NRUH = 3
- RGIF = 1
- Namespace PHNDL 0 maps to RUH 0
- Namespace PHNDL 1 maps to RUH 1
- Namespace PHNDL 2 maps to RUH 2

The host submits a Write with PID `0x8001`.

The controller decodes:

```
PID   = 0x8001
RGID  = 1
PHNDL = 1
RUH   = 1
```

It then looks up RUH 1's reference for RG 1, which in our example is RU-E.

Result: The Write is directed to RU-E.

## 4. What happens when an RU becomes full?

Suppose RUH 1 currently references RU-B in RG 0.

![[ruh rollover.png]]

RUH 1 changes its reference from a full RU to another available RU. Previously written data remains accessible through normal LBA mapping.

The process is:
1. RU-B reaches its writable capacity.
2. The controller identifies an available empty RU, such as RU-G, within RG 0.
3. RUH 1 is updated to reference RU-G.
4. Subsequent writes targeting RG 0 and RUH 1 are directed to RU-G.
5. RU-B remains available for reading its valid data until that data is invalidated or relocated and the RU is reclaimed.


The host can continue using the same PID. It does not need to know the identity of RU-G.

## 5. Relationship summarized

|Characteristic|RG|RUH|RU|
|---|---|---|---|
|What is it?|Reclamation domain|Logical reference resource|Reclaimable storage unit|
|Contains or references|Multiple RUs|One RU per RG|User data|
|Selected by|RGID|Placement Handle mapping|RUH reference|
|Changes as writes occur?|Generally stable configuration|Referenced RU can change|Capacity and state change|
|Host directly identifies it?|Yes, through PID|Indirectly via PHNDL|No|
|Main purpose|Organize reclamation|Direct placement|Store and reclaim data|

For firmware validation, you can inspect the RUH Status management operation to obtain the PID, RUH identifier, Estimated Active Reclaim Unit Time Remaining (EARUTR), and Reclaim Unit Available Media Writes (RUAMW).

The central relationship to remember is:

A Reclaim Group contains Reclaim Units. A Reclaim Unit Handle references one Reclaim Unit in each Reclaim Group. The PID tells the controller which Reclaim Group and which RUH-associated placement path to use.

This abstraction lets the host maintain stable placement identifiers even as the controller changes the underlying Reclaim Units.