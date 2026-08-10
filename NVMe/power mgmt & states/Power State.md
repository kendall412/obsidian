#power_state
### Check current power state
```bash
sudo nvme get-feature /dev/nvme0 -f 2 -H
```

### Power State Descriptor 
#power_state_descriptor
![[fig.626 power state descriptor.png]]
### Change power state
Run the `set-feature` command and supply the desired power state number (e.g., `0` for maximum performance, or a higher number like `3` or `4` for lower power consumption):
```bash
sudo nvme set-feature /dev/nvme0 -f 2 -v <state_number>
```

- **Example (Max Performance / PS0):** `sudo nvme set-feature /dev/nvme0 -f 2 -v 0`
- **Example (Power Saving / PS4):** `sudo nvme set-feature /dev/nvme0 -f 2 -v`

### Make the change persistent (optional)
Manual power state changes reset after a reboot. To force a low-power or high-performance state permanently on Linux, you can create a simple `systemd` service: [1](https://forum.45homelab.com/t/nvme-micron-7450-how-to-reduce-power-consumption/3717), [2](https://forum.45homelab.com/t/nvme-micron-7450-how-to-reduce-power-consumption/3717)

1. Open a new service file
```bash
sudo nano /etc/systemd/system/force-nvme-power.service
```

2. Paste the following configuration
```text
[Unit]
Description=Set NVMe Power State on Boot
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/nvme set-feature /dev/nvme0n1 -f 2 -v 0

[Install]
WantedBy=multi-user.target
```

3. Enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now force-nvme-power.service
```

