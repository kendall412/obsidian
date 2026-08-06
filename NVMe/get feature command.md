## Get Feature Command

To use the get-feature command in nvme-cli, execute the basic syntax sudo nvme `get-feature <device> --feature-id=<fid>`.

```
sudo nvme get-feature <device> [options]
```

### Essential Parameters & Flags<br>
- `<device>`: Mandatory. The NVMe character device (e.g., `/dev/nvme0`) or namespace block device (e.g., `/dev/nvme0n1`).
- `-f <fid>` / --feature-id=<fid>: Mandatory. The ID of the feature you want to fetch, provided in hexadecimal format.
- `-H / --human-readable`: Parses the bit fields into a readable layout rather than raw numbers.
- `-n <nsid> / --namespace-id=<nsid>`: Specifies the target namespace (only needed if a feature applies to a specific namespace rather than the entire controller).
- `-s <select> / --sel=<select>`: Selects which value to return:
    - 0 = Current
    - 1 = Default
    - 2 = Saved
    - 3 = Supported capabilities

#### Example:
1. Check Power Management StateTo view the current power state configuration on /dev/nvme0, run:<br>
```
sudo nvme get-feature /dev/nvme0 -f 0x02
```

2. Get the Number of Configured Queues (Human Readable)<br>
To check how manysubmission and completion queues are allocated on the drive, use the -H flag:
```
sudo nvme get-feature /dev/nvme0 -f 0x07 -H
```

