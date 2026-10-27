# 02 — Build B: Debian on SSD, Preserve HDD

**Outcome:** Debian 13 runs on the 128 GB SSD. The original 1 TB HDD remains intact and is later exported only to VM1 as an iSCSI LUN.

## Safe build sequence

1. Verify an offline backup and representative restore of required HDD data.
2. Power down and physically disconnect the HDD.
3. Install Debian on the SSD only: single ext4 root plus swap, SSH, no desktop.
4. Configure storage-b as 192.168.1.51/24 with gateway 192.168.1.254; verify SSH and updates.
5. Power down, reconnect the HDD, boot from the SSD, then inspect both disks by model, UUID, and mountpoint.
6. Mount the old ext4 partition read-only with no journal replay only for inspection. Unmount it before target setup.

The old HDD is one ext4 root tree; obsolete Debian files remain until migration/backup validation. Do not delete directories to create a fictitious data partition. Record the filesystem UUID and partition PARTUUID from the actual HDD before continuing to [03](03-build-b-iscsi-target.md).

## Boundary

After target configuration, Build B owns the disk and LIO service; VM1 owns the mounted filesystem. Build B must not locally mount the ext4 partition while VM1 has an iSCSI session.
