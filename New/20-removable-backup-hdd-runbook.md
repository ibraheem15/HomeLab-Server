# 20 — Removable Backup HDD Runbook

## Physical and virtual path

```text
500 GB USB HDD → Build A USB port → Proxmox USB passthrough → services-vm → ext4 → /mnt/backup-seagate → Backrest /repo
```

The disk is attached to Build A but mounted only inside VM1. Once Proxmox USB passthrough is configured for the device/port, ordinary backup sessions require no Proxmox edits.

Stable filesystem identity:
Find the uuid:
```bash
ls -l /dev/disk/by-uuid | grep ea3f8b0d
```

```text
UUID=ea3f8b0d-<SECRET>-e4029de67a34
```

Never automate against `/dev/sdb1` or `/dev/sdc1`; USB/SCSI letters can change after every reconnect.

## VM1 fstab policy

The external disk must not auto-mount during normal VM operation:

```fstab
UUID=ea3f8b0d-<SECRET>-e4029de67a34 /mnt/backup-seagate ext4 noauto,nofail,x-systemd.device-timeout=15s 0 2
```

Create the mountpoint once:

```bash
sudo mkdir -p /mnt/backup-seagate
```

## Automation script

Install the wiki's `backup-usb.sh` inside VM1:

```bash
sudo install -m 0750 backup-usb.sh /usr/local/sbin/backup-usb
```

The script provides:

```text
backup-usb start   detect UUID → verify iSCSI → mount HDD → verify repo → recreate Backrest
backup-usb status  report device, mounts, container, and active Restic process
backup-usb stop    refuse active Restic → stop Backrest → sync → unmount → declare safe
```

It also serializes operations with `/run/lock/backup-usb.lock`, validates the repository `config` and `is_mounted` sentinel, and checks that both iSCSI and USB filesystems are read-write.

## Normal backup session

1. Attach or power on the USB HDD on Build A.
2. Inside VM1 run `sudo backup-usb start`.
3. Start the required plan in Backrest and wait for a successful completion.
4. Run `sudo backup-usb status`; no Restic process should be active.
5. Run `sudo backup-usb stop`; disconnect only after `SAFE TO REMOVE`.

The script manages storage/container lifecycle; it deliberately does not select or trigger a Backrest plan.

## Why Backrest is recreated after mounting

Docker bind mounts are normally `rprivate`. If Backrest starts while `/mnt/backup-seagate` is only an empty directory and the USB filesystem is mounted later, the running container may continue to see the empty directory.

Correct order:

```text
mount USB filesystem → recreate Backrest → verify /repo/config inside container
```

This prevents the “new repository will be initialized” failure.

## Initial disk inspection and repair

The disk was first mounted read-only with journal replay disabled:
Find the uuid:
```bash
ls -l /dev/disk/by-uuid | grep ea3f8b0d
```
```bash
sudo mount -o ro,noload \
  /dev/disk/by-uuid/ea3f8b0d-<SECRET>-e4029de67a34 \
  /mnt/backup-seagate
```

The read-only `e2fsck -f -n` reported free block/inode count mismatches because journal recovery was skipped. A read-only check diagnoses but cannot repair.

For repair, the filesystem must be unmounted and not held by any process:

Find the uuid:
```bash
ls -l /dev/disk/by-uuid | grep ea3f8b0d
```

```bash
sudo umount /mnt/backup-seagate
sudo e2fsck -f -p \
  /dev/disk/by-uuid/ea3f8b0d-<SECRET>-e4029de67a34
```

Exit codes: `0` = clean, `1` = corrected, `2` = corrected/reboot advisable, `4+` = unresolved/error. Never run repair against a mounted filesystem.

## Failure handling

If `backup-usb start` cannot detect the UUID within 60 seconds, inspect Proxmox USB passthrough, cable/power, and whether the enclosure or physical port changed. Do not substitute a guessed device path.

If `backup-usb stop` detects Restic, wait. If unmount reports busy, identify the owner:

```bash
sudo fuser -vm /mnt/backup-seagate
```

Do not unplug, force-unmount, or power off until the mount is gone:

```bash
findmnt /mnt/backup-seagate
```

No output = unmounted. The powered-off interval is part of the backup threat model: it limits ransomware, operator-error, and online-corruption exposure.
