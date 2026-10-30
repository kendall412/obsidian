
> Reclaim Unit Handle Update is an NVMe Flexible Data Placement (FDP) **management operation that allows the host to request that the controller stop using the currently associated Reclaim Unit (RU) and associate the Reclaim Unit Handle (RUH) with a new RU.** It is performed using the I/O Management Send command, with the Reclaim Unit Handle Update management operation.

### 1. What exactly does it update?

The operation updates the RU associated with an RUH, not the RUH identifier itself.

For example, assume:
- Reclaim Group ID = 0
- Placement Handle = 1
- Placement Handle 1 maps to RUH 2
- RUH 2 currently references RU-A

![[before and after ruh update.png]]

### 2. What happens internally?

When the controller processes a successful RUH Update:
1. The host identifies the PID whose associated RUH should be updated.
2. The controller resolves the PID to the Reclaim Group and RUH.
3. The controller ends the current RU association for new placement.
4. The controller associates the RUH with another available RU in that Reclaim Group.
5. Subsequent writes using the same PID are directed to the newly associated RU.
    
The previous RU may still contain valid data. That data remains accessible through its LBAs. RUH Update does not erase, deallocate, or relocate the data already stored in the previous RU.

### 3. Why would the host issue RUH Update?

The primary reason is to separate data with different expected lifetimes.

Suppose an application writes data that belongs to two generations:

|Data|PID|Desired RU|
|---|---|---|
|Generation A|0x0001|RU-A|
|Generation B|0x0001|RU-B|

The host can:

1. Write Generation A using PID `0x0001`.
    
2. Issue RUH Update for PID `0x0001`.
    
3. Write Generation B using the same PID.
    
4. The controller places Generation B in the newly associated RU.
    

This allows the host to establish a new placement boundary without creating another PID or RUH.

### 4. How is it different from automatic RU rollover?

|Characteristic|Automatic RU rollover|RUH Update|
|---|---|---|
|Initiated by|Controller|Host|
|Typical trigger|Active RU runs out of writable capacity|Host requests a new placement boundary|
|Requires RU to be full?|Generally yes|No|
|RUH identifier changes?|No|No|
|Previously written data moved?|No|No|
|Future writes use another RU?|Yes|Yes, following successful update|

### 5. Example using nvme-cli

```
# Check RUH status before the update
sudo nvme fdp status /dev/nvme0 -n 1

# Request RUH Update for PID 1
sudo nvme fdp update /dev/nvme0 -n 1 -p 1

# Check RUH status afterward
sudo nvme fdp status /dev/nvme0 -n 1
```

You can compare the RUAMW (Reclaim Unit Available Media Writes) field before and after the operation.

If the update associates a fresh RU, RUAMW would generally reflect the newly available writable capacity. The exact value and timing depend on the controller implementation.

### 6. Important considerations for validation

- An RUH Update is a request to the controller, not a command to select a specific physical NAND block.
    
- The host specifies PIDs, not physical RU addresses.
    
- An update may leave unused capacity in the previous RU.
    
- A successful command does not mean garbage collection has occurred.
    
- RUH Status provides logical status information, not necessarily the physical identity of the new RU.
    
- The controller must follow the applicable FDP configuration and RUH update rules.
    

In summary: Reclaim Unit Handle Update allows the host to deliberately end the current placement association and begin using another Reclaim Unit through the same RUH and PID. Its main benefit is separating data into different reclamation units to improve data lifetime management and potentially reduce write amplification.