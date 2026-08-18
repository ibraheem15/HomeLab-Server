# 02 — Build B: Debian SSD Boot While Preserving the HDD

## Goal

Move Debian from the only existing HDD to a newly found 120 GB SSD without wiping or repartitioning the HDD. Build B becomes storage-only; all compute-heavy containers move to Build A.

## Why the HDD was not reformatted

The original HDD had one large ext4 root partition containing both Debian system files and application data:

```text
sda1  vfat   /boot/efi
sda2  ext4   /
sda3  swap
```

There was no separate “data partition.” A partition is a block-device region, not “files excluding Debian.” Deleting old OS directories manually would not turn the existing root partition into a new partition, and deleting system directories prematurely could destroy useful application configuration.

The adopted approach:

1. Keep a verified external restic backup.
2. Install fresh Debian to the new SSD with the HDD physically disconnected.
3. Reconnect the HDD.
4. Inspect it read-only.
5. Export the existing ext4 partition as the iSCSI LUN.
6. Leave obsolete Debian files in place until migration and backup validation are complete.

## Phase 3 — Backup before hardware changes

The old Backrest schedule had previously been found disabled. Before changing disks:

1. Enable/confirm the schedule.
2. Power on the external backup HDD.
3. Run a full backup of the original disk’s required data.
4. Run repository integrity checks and representative restores/checksums.
5. Power the backup HDD off again.

Never treat same-disk snapshots, an iSCSI LUN, or an old copy on the same HDD as backup.

## Phase 4A — Install Debian on the 120 GB SSD

1. Power Build B off.
2. Disconnect the original HDD.
3. Connect only the 120 GB SSD and Debian USB.
4. Boot the Debian installer.
5. Select plain **Install** if graphical mode fails.
6. Hostname: `storage-b`.
7. Use guided partitioning on the SSD; single ext4 root + swap.
8. Install SSH server and standard system utilities; no desktop environment.
9. Reboot and verify the root filesystem is the SSD.

Initial verification:

```bash
hostnamectl
findmnt /
lsblk -o NAME,SIZE,FSTYPE,UUID,MOUNTPOINTS,MODEL
sudo swapon --show
```

Expected key result:

```text
/      /dev/sda1  ext4
swap   /dev/sda5
```

The non-root `hostnamectl` warnings about product UUID and hardware serial access were permission warnings, not installation failures. Running `sudo hostnamectl` displayed the protected fields.

Enable SSH:

```bash
sudo systemctl enable --now ssh
systemctl is-enabled ssh
systemctl is-active ssh
```

Configure `<STORAGE_SERVER_LAN_IP>/24`, gateway/DNS `<LAN_GATEWAY_IP>`, then install Tailscale.

## Phase 4B — Reconnect and inspect the HDD

Power down, reconnect the HDD, and boot from the SSD. Verify by model and mountpoint; device letters can change.

Observed state:

```text
sda  119.2G  Lexar SSD     new Debian root
sdb  931.5G  WDC HDD       preserved old disk
```

Mount the old ext4 partition read-only without journal replay:

```bash
sudo mkdir -p /mnt/old-hdd
sudo mount -t ext4 -o ro,noload \
  UUID=<FILESYSTEM_UUID> \
  /mnt/old-hdd
```

Verify:

```bash
findmnt /mnt/old-hdd
df -hT /mnt/old-hdd
sudo ls -lah /mnt/old-hdd
```

The old root tree—including `boot`, `etc`, `home`, `docker`, `opt`, and `var`—was intact. Usage was approximately 447 GiB of 915 GiB formatted capacity.

Unmount before checking or exporting:

```bash
cd ~
sudo umount /mnt/old-hdd
```

Run a non-modifying ext4 check during initial inspection:

```bash
sudo e2fsck -f -n \
  /dev/disk/by-uuid/<FILESYSTEM_UUID>
```

The check completed all five passes. Three messages said an extent tree “could be shorter”; these were optimization suggestions, not corruption. Because `-n` was used, nothing changed.

Capture stable identity:

```bash
sudo blkid /dev/sdb2
ls -l /dev/disk/by-partuuid/
```

Authoritative partition path:

```text
/dev/disk/by-partuuid/<PARTITION_UUID>
```

## Final Build B rule

Once the partition is exported and VM1 mounts it, Build B must not mount `/dev/sdb2`. Build B owns the physical disk and target service; VM1 owns the filesystem.

