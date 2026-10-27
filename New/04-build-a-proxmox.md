# 04 — Build A: Proxmox and Disk Ownership

Install current Proxmox VE using the [official guide](https://pve.proxmox.com/pve-docs/). pve-a is 192.168.1.10/24 with gateway 192.168.1.254.

## Storage decision

| Disk | Owner | Use |
|---|---|---|
| 500 GB NVMe | Proxmox | host, VM OS disks, databases, configuration |
| 1 TB SATA SSD | TrueNAS VM 101 | whole-disk passthrough, ZFS pool sata |

Install Proxmox on the NVMe only. Capture model, serial, by-id path, and SMART information for the SATA SSD, then pass the entire disk to TrueNAS. Do not create Proxmox storage, a filesystem, or a mount on that disk.

Current LVM allocation: root 64 GiB, swap 8 GiB, thin pool about 347 GiB, and about 50 GiB unallocated. Monitor thin-pool data/metadata; thin provisioning does not create capacity.

## Operations

Use the release-matched [Proxmox package-repository guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html) for updates. Tailscale is administration/recovery only; retain console access. Do not install or configure open-iscsi on the host: VM1 is deliberately the sole initiator for Build B.
