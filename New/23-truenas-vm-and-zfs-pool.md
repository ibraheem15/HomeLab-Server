# 23 — TrueNAS VM and ZFS Pool

VM 101: 2 host-type vCPU, 8 GiB fixed RAM, local virtual boot disk, LAN 192.168.1.25, and storage bridge 10.20.0.10 without gateway. Attach the Samsung SATA SSD as a whole raw disk by stable by-id path; TrueNAS owns it exclusively.

## Build sequence

1. Create the VM with UEFI/Q35, boot from current TrueNAS installer, and install to its virtual boot disk.
2. Attach the whole physical SATA SSD only after verifying model/serial/by-id; do not attach a Proxmox-formatted volume.
3. Configure both addresses; storage bridge has no default gateway.
4. In TrueNAS, create pool sata from the identified SATA SSD. This erases that disk: require explicit stable-ID confirmation first.
5. Confirm pool health, SMART visibility, and boot/import persistence.

Serial-console/display quirks during installer are VM console issues, not permission to change disk layout. Start TrueNAS before VM2/VM3 SMB consumers. Use [TrueNAS documentation](https://www.truenas.com/docs/) for release-specific screens.
