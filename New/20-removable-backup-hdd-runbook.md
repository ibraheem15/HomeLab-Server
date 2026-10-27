# 20 — Removable Backup HDD

The external Restic repository disk passes through Proxmox to VM1 and is normally disconnected/off. Identify it by UUID and model, never sdX.

## Session

    lsblk -f
    sudo blkid
    findmnt /mnt/backup-seagate

1. Attach the expected disk; verify identity and repository files.
2. Mount it at the documented repository path.
3. Start/recreate Backrest only after the mount exists.
4. Run backup and wait for Restic children to finish.
5. Stop through backup-usb stop, sync, unmount, and disconnect only after SAFE TO REMOVE.

Backrest must not run against an empty mountpoint: it can offer to initialize a wrong repository. A busy unmount means a shell or process holds the mount; inspect it rather than force-removing the disk:

    sudo fuser -vm /mnt/backup-seagate

Filesystem repair requires all writers stopped, a fully unmounted partition, exact device identity, and offline repair. See [19](19-backrest-split-storage-backup.md) for backup content.
