# 22 — Build A Storage Architecture

The NVMe is for Proxmox, VM OS disks, application configuration, and databases. The whole SATA SSD belongs to TrueNAS VM 101 as pool sata. VM2 and VM3 consume only child datasets through SMB on the isolated storage bridge.

TrueNAS is used because its data lifecycle is separate from Proxmox VM disks and needs dataset quotas/ACLs/SMB. SMB suits VM2/VM3 bulk files; it is not a database location. The pool is one disk with no redundancy, so monitor SMART/capacity and maintain offline backup. Do not share the pool root or mount the SSD on Proxmox.
