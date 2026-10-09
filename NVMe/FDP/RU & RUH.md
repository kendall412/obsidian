
In NVMe Flexible Data Placement (FDP), a Reclaim Unit (RU) is a logical unit of storage that the controller manages and reclaims, while a Reclaim Unit Handle (RUH) is a logical reference used to direct writes to a Reclaim Unit.

The key relationship is:
A Reclaim Unit Handle identifies the write-placement destination, while the Reclaim Unit is where the data is actually stored.

### 1. Relationship between RU and RUH

|                     | Reclaim Unit (RU)                               | Reclaim Unit Handle (RUH)         |
| ------------------- | ----------------------------------------------- | --------------------------------- |
| Purpose             | Stores data and serves as a unit of reclamation | Directs writes to an RU           |
| Type                | Logical storage allocation                      | Logical handle                    |
| Contains user data? | Yes                                             | No                                |
| Managed by          | SSD controller                                  | SSD controller                    |
| Changes over time?  | Filled, closed, reclaimed                       | Can be associated with another RU |
| Used in PID?        | Not directly                                    | Yes, through RUH identifier       |


```
Reclaim Group 0
|
+-- RU Handle 0
|       |
|       +----> Reclaim Unit A
|               +-- Data
|               +-- Data
|               +-- Free Capacity
|
+-- RU Handle 1
|       |
|       +----> Reclaim Unit B
|               +-- Data
|               +-- Free Capacity
|
+-- RU Handle 2
        |
        +----> Reclaim Unit C
                +-- Data
                +-- Free Capacity
```

In this example:
- RUH 0 directs writes to Reclaim Unit A.
- RUH 1 directs writes to Reclaim Unit B.
- RUH 2 directs writes to Reclaim Unit C.


### 3. What happens when a Reclaim Unit becomes full?
Suppose the host continues writing data using RUH 0.

```
STEP 1: Initial State

RUH 0 ------> Reclaim Unit A
              [Data][Data][Free]


STEP 2: More Writes

RUH 0 ------> Reclaim Unit A
              [Data][Data][Data]
                     FULL


STEP 3: Controller Allocates Another RU

RUH 0 ------> Reclaim Unit D
              [Free][Free][Free]

Reclaim Unit A
              [Data][Data][Data]
                     FULL
```

The controller can associate RUH 0 with another Reclaim Unit when the previous unit becomes full.

Notice that:
1. The host continues using RUH 0.
2. The controller changes the underlying RU association.
3. The host does not need to select the physical storage location.

The full Reclaim Unit A can later be reclaimed when its valid data has been relocated or invalidated.

### 4. Why does FDP use Reclaim Unit Handles?
Without this abstraction, the host would need to understand and manage individual Reclaim Units. Instead, *FDP allows the host to specify a placement destination while leaving storage allocation and reclamation under controller management.*

For example:
```
Host
 |
 +-- Write Command
 |      |
 |      +-- PID
 |           +-- RGID = 0
 |           +-- RUHID = 2
 |
 v
SSD Controller
 |
 +-- Reclaim Group 0
 |      |
 |      +-- RU Handle 2
 |             |
 |             v
 |         Reclaim Unit C
 |             |
 |             v
 |         NAND Storage
```

The host selects the placement using the PID. The controller resolves that placement to the currently associated Reclaim Unit.

### Can one RUH be associated with multiple Reclaim Units?
A useful distinction is between the current write destination and previously written Reclaim Units. *A RUH may be associated with different Reclaim Units over time.* Previously filled units can still contain valid data after the handle moves to a new unit.

Therefore, *one RUH can be responsible for a sequence of Reclaim Units over its lifetime,* but this does not mean every one of those units is simultaneously the current write destination.

In summary:
- Reclaim Unit: The logical storage unit where data is placed and later reclaimed.
- Reclaim Unit Handle: The logical reference through which the host directs writes to that storage.
- Placement Identifier: Identifies the Reclaim Group and RU Handle used for write placement.

Think of the RUH as a reusable pointer and the RU as the storage unit that pointer currently targets.