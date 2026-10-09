In NVMe FDP, Host Specified RUH and Controller Specified RUH describe who selected the Reclaim Unit Handle (RUH) for a namespace.

The distinction is reported in the Reclaim Unit Handle Usage Log Page (LID 21h) through the RUHA field.

## 1. Host Specified RUH (RUHA = 01h)

The host explicitly requests which RUHs a namespace should use during namespace creation.

For example, assume an FDP configuration supports eight RUHs (0–7).

The host creates Namespace 1 and requests RUH 2 and RUH 4 through the Placement Handle List.

```
HOST
 |
 +-- Create Namespace 1
 |      |
 |      +-- Placement Handle List
 |             +-- RUH 2
 |             +-- RUH 4
 |
 v
SSD CONTROLLER
 |
 +-- Namespace 1
        |
        +-- RUH 2 ---> Reclaim Units
        |
        +-- RUH 4 ---> Reclaim Units
```

The controller reports RUH 2 and RUH 4 as Host Specified (RUHA = 01h).

This allows the host to explicitly choose placement handles for its workload.

## 2. Controller Specified RUH (RUHA = 02h)

The host does not explicitly request a Placement Handle List during namespace creation.

Instead, the controller selects the default RUH used by that namespace.

```
HOST
 |
 +-- Create Namespace 2
 |      |
 |      +-- No Placement Handle List
 |
 v
SSD CONTROLLER
 |
 +-- Select Default RUH
 |      |
 |      +-- RUH 0
 |
 +-- Namespace 2
        |
        +-- RUH 0 ---> Reclaim Units
```

The controller reports RUH 0 as Controller Specified (RUHA = 02h).

The host did not choose the RUH; the controller selected it.

## 3. Comparison

|Characteristic|Host Specified|Controller Specified|
|---|---|---|
|RUHA value|`01h`|`02h`|
|Who selects the RUH?|Host|Controller|
|Selection mechanism|Placement Handle List|Default RUH|
|Selected during|Namespace creation|Namespace creation|
|Host controls handle selection?|Yes|No|

## 4. Example RUH Usage Log Page

Suppose the controller supports eight RUHs, and the namespaces have been configured as described above.

|RUH ID|RUHA|Meaning|
|---|---|---|
|0|`02h`|Controller Specified|
|1|`00h`|Unused|
|2|`01h`|Host Specified|
|3|`00h`|Unused|
|4|`01h`|Host Specified|
|5|`00h`|Unused|
|6|`00h`|Unused|
|7|`00h`|Unused|

This tells the host which handles were explicitly requested and which handle was selected by the controller.

## 5. Important distinction: Who selects the RUH versus who manages the RU

Even when the host specifies a RUH, the SSD controller still manages the actual Reclaim Units.

For example:

```
Host Specifies RUH 2
         |
         v
SSD Controller
         |
         +-- RUH 2
               |
               v
           Reclaim Unit A
               |
               | RU becomes full
               v
           Reclaim Unit B
```

The host selects RUH 2, but the controller determines which underlying Reclaim Unit is associated with that handle.

Therefore, Host Specified does not mean the host controls physical NAND allocation or reclaim operations.

### Key takeaway

- Host Specified RUH: The host explicitly requests a particular RUH for a namespace.
    
- Controller Specified RUH: The controller selects the default RUH for a namespace.
    
- Both: The controller remains responsible for allocating, managing, and reclaiming the underlying Reclaim Units.
    

The distinction concerns who selected the logical placement handle, not who manages the physical storage.