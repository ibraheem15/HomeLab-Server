# 27 — VM3 Checkpoint

VM 104 has Debian installed: 4 host-type vCPU, 4 GiB RAM, 30 GiB local disk, intended LAN 192.168.1.40/24, and storage 10.20.0.30/24 with no gateway. It is not operational yet.

## Finish in order

1. Confirm the second VirtIO NIC exists in Proxmox and guest.
2. Apply static LAN/default route and storage-only address.
3. Install cifs-utils; create /mnt/vm3-extra and a root-readable SMB credentials file.
4. Manually mount //10.20.0.10/vm3-extra, test read/write, unmount.
5. Add a systemd-aware fstab automount and reboot-test it.
6. Install Tailscale and owner workload only after networking/storage pass.

Keep databases, configuration, host keys, and mutable state on VM3 local disk. SMB is bulk data only. VM3’s final Tailscale organization/owner policy remains undecided.
