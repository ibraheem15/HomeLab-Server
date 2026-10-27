# 05 — VM1: Services Guest

VM 102 (services-vm) runs Debian 13 with 6 host-type vCPU, 24 GiB fixed RAM, a 100 GiB local-lvm disk, OVMF/Q35, and a VirtIO LAN NIC at 192.168.1.20/24.

## Ownership

The local VM disk owns Docker/Compose definitions, application configuration, Frigate SQLite, Immich PostgreSQL, and all mutable databases. Build B iSCSI at /mnt/storage-b is bulk media only. This separation is mandatory because storage/network failure must not corrupt databases.

Install Debian with static networking, SSH, current updates, Docker Engine, and Compose using [Docker’s Debian guide](https://docs.docker.com/engine/install/debian/). Confirm the Proxmox NIC has no VLAN tag on the untagged LAN.

If VM 102 was accidentally converted to a template, convert it back before booting or editing it. GPU passthrough is deferred: Frigate uses CPU/OpenVINO. Do not add a DRM device mapping until the iGPU has been intentionally passed through and verified.
