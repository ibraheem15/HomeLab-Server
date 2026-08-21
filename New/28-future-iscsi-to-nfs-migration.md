# 28 — Future Build B Migration: iSCSI to NFS

**Status:** Deferred. The current iSCSI setup is operational and must remain unchanged until this runbook is deliberately scheduled.  
**Objective:** Let Build B own and mount its physical ext4 HDD locally, then export the existing files directly to `services-vm` through NFSv4.  
**Data migration:** None. The existing partition remains ext4; no wipe, reformat, or file copy is required.

## Why this migration may be useful

Current path:

```text
Build B HDD
→ LIO/targetcli raw block export
→ iSCSI session
→ VM1 block device
→ VM1 mounts ext4 at /mnt/storage-b
→ Immich/Frigate/Jellyfin
```

Proposed path:

```text
Build B HDD
→ Build B mounts ext4 at /srv/storage-b
→ NFSv4 export
→ VM1 mounts NFS at /mnt/storage-b
→ Immich/Frigate/Jellyfin
```

NFS moves filesystem ownership to the machine physically holding the HDD. A Build B restart then interrupts a file share rather than removing a live block device from beneath VM1. Build B performs ext4 journal recovery locally; VM1's hard NFS mount waits and reconnects.

This is a reliability simplification, not an urgent repair and not guaranteed to improve throughput. Both protocols remain limited by the same HDD and 1 GbE LAN.

## Current identifiers

```text
Build B hostname:   storage-b
Build B LAN IP:     192.168.1.51
VM1 hostname:       services-vm
VM1 LAN IP:         192.168.1.20
Target IQN:         iqn.2026-08.arpa.home:storage-b.media
HDD PARTUUID:       b08c7158-f19f-4a1f-b13c-f839eab06a67
ext4 UUID:          675a9a2e-c066-40ae-8d92-ff0a0c4ec61e
Current VM1 mount:  /mnt/storage-b
Future Build B mount: /srv/storage-b
```

## Non-negotiable safety rules

1. Never mount the HDD on Build B while VM1 has the iSCSI LUN mounted or logged in.
2. Stop every writer, flush writes, unmount VM1, and log out of iSCSI before releasing the target backstore.
3. Run `e2fsck` only after confirming the target no longer holds the block device.
4. Keep the targetcli JSON backup until NFS has passed reboot, write, Immich, and Frigate tests.
5. Never use an NFS `soft` mount for writable application media.

## Storage placement after migration

| Workload | Location |
|---|---|
| Immich PostgreSQL | VM1 local NVMe virtual disk |
| Vaultwarden SQLite | VM1 local NVMe virtual disk |
| Frigate SQLite/config | VM1 local disk under `/config` |
| Frigate `/tmp/cache` | VM1 tmpfs |
| Immich media library | Build B NFS |
| Frigate recordings | Build B NFS |
| Jellyfin media | Build B NFS |
| TrueNAS datasets | Continue serving VM2/VM3 only; not in this path |

## Phase 0 — Backup and maintenance window

Before touching storage:

- Run and verify the external Backrest/restic backup.
- Confirm a current Immich PostgreSQL dump exists.
- Schedule downtime for every container using `/mnt/storage-b`.
- Keep local console or Proxmox console access to VM1 in case SSH blocks on storage I/O.

Identify affected containers inside VM1:

```bash
sudo docker inspect $(sudo docker ps -q) \
  --format '{{.Name}} {{range .Mounts}}{{.Source}} -> {{.Destination}} {{end}}' |
grep '/mnt/storage-b'
```

Record the working state:

```bash
findmnt /mnt/storage-b
sudo iscsiadm -m session -P 1
grep -n '/mnt/storage-b' /etc/fstab
```

On Build B, preserve the target definition:

```bash
sudo cp \
  /etc/rtslib-fb-target/saveconfig.json \
  /etc/rtslib-fb-target/saveconfig.json.before-nfs
```

## Phase 1 — Stop VM1 writers

Stopping Docker entirely is the safest migration boundary:

```bash
sudo systemctl stop docker.service docker.socket
cd /
sudo fuser -vm /mnt/storage-b
findmnt -R /mnt/storage-b
```

Unmount any child or convenience bind mounts first. Then flush and detach the main filesystem:

```bash
sync
sudo umount /mnt/storage-b
findmnt /mnt/storage-b
```

`findmnt` must return nothing before continuing.

## Phase 2 — Disable iSCSI login and retry automation

Inside VM1:

```bash
sudo iscsiadm -m node \
  -T iqn.2026-08.arpa.home:storage-b.media \
  -p 192.168.1.51:3260 \
  --op update \
  -n node.startup \
  -v manual

sudo iscsiadm -m node \
  -T iqn.2026-08.arpa.home:storage-b.media \
  -p 192.168.1.51:3260 \
  --logout

sudo systemctl disable --now storage-b-ready.service 2>/dev/null || true
sudo iscsiadm -m session
```

Required result:

```text
iscsiadm: No active sessions
```

Do not uninstall `open-iscsi`; keeping it makes rollback easier.

## Phase 3 — Release the HDD on Build B

Disable the saved target from automatically returning and stop its active kernel configuration:

```bash
sudo systemctl disable --now rtslib-fb-targetctl.service
```

Verify that nothing still exposes or holds the partition:

```bash
systemctl is-active rtslib-fb-targetctl.service
sudo ss -lntp | grep ':3260'
sudo fuser -v \
  /dev/disk/by-partuuid/b08c7158-f19f-4a1f-b13c-f839eab06a67
```

Required state:

```text
rtslib-fb-targetctl: inactive
TCP 3260: no listener
fuser: no users
```

If the partition remains busy, stop. Never mount it locally until the target has released it.

## Phase 4 — Offline ext4 check

On Build B:

```bash
sudo e2fsck -f -p \
  /dev/disk/by-partuuid/b08c7158-f19f-4a1f-b13c-f839eab06a67

echo "e2fsck exit code: $?"
```

| Exit | Action |
|---:|---|
| `0` | Clean; continue |
| `1` | Errors corrected; continue |
| `2` | Reboot Build B before continuing |
| `4+` | Stop and investigate |

## Phase 5 — Mount the HDD locally on Build B

```bash
sudo mkdir -p /srv/storage-b
```

Add to Build B `/etc/fstab`:

```fstab
UUID=675a9a2e-c066-40ae-8d92-ff0a0c4ec61e /srv/storage-b ext4 defaults,noatime,nofail,x-systemd.device-timeout=60s 0 2
```

Apply and verify:

```bash
sudo systemctl daemon-reload
sudo mount /srv/storage-b
findmnt /srv/storage-b
df -hT /srv/storage-b
sudo ls -lah /srv/storage-b/home/ibraheem
```

Required: source is the HDD UUID, filesystem is `ext4`, and options contain `rw`.

## Phase 6 — Configure the NFS server

On Build B:

```bash
sudo apt update
sudo apt install nfs-kernel-server
```

Add to `/etc/exports`:

```exports
/srv/storage-b 192.168.1.20(rw,sync,no_subtree_check,no_root_squash,fsid=0)
```

Initial `no_root_squash` preserves compatibility with containers that write as root. The export is restricted to VM1's static LAN address. After migration, tighten exports and UID/GID ownership if all containers can run without root mapping.

Prevent NFS from exporting an empty directory on Build B's SSD when the HDD is missing. Create:

```text
/etc/systemd/system/nfs-server.service.d/storage-b.conf
```

Contents:

```ini
[Unit]
RequiresMountsFor=/srv/storage-b

[Service]
ExecStartPre=/usr/bin/mountpoint -q /srv/storage-b
```

Apply:

```bash
sudo systemctl daemon-reload
sudo exportfs -rav
sudo systemctl enable --now nfs-server.service
```

Verify:

```bash
systemctl status nfs-server.service --no-pager
sudo exportfs -v
sudo ss -lntp | grep ':2049'
```

## Phase 7 — Mount NFS on VM1

Inside VM1:

```bash
sudo apt update
sudo apt install nfs-common
nc -vz 192.168.1.51 2049
```

Comment out the old ext4/iSCSI `/etc/fstab` entry. Do not delete it until the rollback window has closed:

```fstab
# UUID=675a9a2e-c066-40ae-8d92-ff0a0c4ec61e /mnt/storage-b ext4 ...
```

Add the NFSv4 mount:

```fstab
192.168.1.51:/ /mnt/storage-b nfs4 rw,hard,proto=tcp,vers=4.2,_netdev,nofail,x-systemd.automount,x-systemd.after=network-online.target,x-systemd.mount-timeout=300s,timeo=600,retrans=2 0 0
```

Apply:

```bash
sudo systemctl daemon-reload
sudo mount /mnt/storage-b
findmnt /mnt/storage-b
df -hT /mnt/storage-b
```

Required source and type:

```text
SOURCE: 192.168.1.51:/
FSTYPE: nfs4
OPTIONS: rw,hard,...
```

`timeo=600` is a 60-second NFS RPC timeout. Because the mount is `hard`, this is not a total recovery limit; the client continues retrying. Never replace `hard` with `soft` for this writable storage.

## Phase 8 — Data and write validation

Inside VM1:

```bash
ls -lah /mnt/storage-b/home/ibraheem

test -d /mnt/storage-b/home/ibraheem/immich-app/library &&
echo "Immich path OK"

test -d /mnt/storage-b/home/ibraheem/camera/frigate/storage &&
echo "Frigate path OK"

sudo sh -c 'echo nfs-test > /mnt/storage-b/.nfs-write-test'
sudo cat /mnt/storage-b/.nfs-write-test
sudo rm /mnt/storage-b/.nfs-write-test
```

No Docker service should start until these checks pass.

## Phase 9 — Docker mount guards

Every Build B-backed Compose mount should use long bind syntax:

```yaml
volumes:
  - type: bind
    source: /mnt/storage-b/path/to/data
    target: /container/path
    bind:
      create_host_path: false
```

This prevents Docker from silently creating missing source directories on VM1's local 100 GB disk.

Keep local application state local:

```text
Frigate /config + frigate.db → VM1
Frigate /tmp/cache           → VM1 tmpfs
Immich PostgreSQL            → VM1
Immich media                 → NFS
```

## Phase 10 — Restart and functional tests

```bash
sudo systemctl start docker.service
sudo docker ps
```

Validate:

- Immich opens existing assets and completes a test upload.
- Frigate live view works, creates new recordings, and plays old recordings.
- Jellyfin reads existing media if deployed.
- No service writes unexpected data to VM1's root disk.
- Logs contain no `server not responding`, `stale file handle`, permission, or I/O errors.

Useful checks:

```bash
sudo docker logs --since 10m frigate
sudo docker logs --since 10m immich_server
journalctl --since '-10 minutes' | grep -Ei 'nfs|server not responding|stale|I/O error'
```

## Phase 11 — Recovery testing

Perform tests in increasing risk order:

1. Reboot VM1 while Build B remains online.
2. Stop application writers, reboot Build B, and verify the NFS mount returns.
3. Run Frigate normally and perform a controlled Build B restart while monitoring Frigate, NFS, and Netdata.

After every test:

```bash
findmnt /mnt/storage-b
findmnt -no OPTIONS /mnt/storage-b
sudo touch /mnt/storage-b/.recovery-test
sudo rm /mnt/storage-b/.recovery-test
sudo docker ps
```

Expected NFS outage behavior:

```text
Build B unavailable
→ hard-mounted NFS operations wait
→ Build B mounts ext4 locally and starts NFS
→ VM1 reconnects
→ blocked operations resume or applications retry
```

An outage longer than Frigate's local cache can still lose current recording segments. NFS improves filesystem failure isolation; it does not create storage redundancy.

## Performance acceptance test

Do not accept the migration only because NFS appears mounted. Observe at least 24 hours of normal recording and Immich use.

Acceptance criteria:

| Check | Requirement |
|---|---|
| Sustained NFS writes | At least 3× aggregate camera bitrate |
| Frigate | No missing-frame or recording-write errors attributable to NFS |
| Netdata | No sustained network saturation or abnormal I/O wait |
| Immich | Browsing, uploads, thumbnails, and playback normal |
| Build B restart | Share and applications recover without ext4 repair |
| VM1 root disk | No media written into fallback directories |

Frigate writes new recording segments to cache before moving retained segments to `/media/frigate`; its SQLite database must remain in local `/config` storage.

## Rollback

Rollback remains possible because the ext4 filesystem was not changed.

```text
Stop Docker on VM1
→ unmount NFS
→ stop/disable NFS on Build B
→ unmount /srv/storage-b on Build B
→ restore/enable rtslib-fb-targetctl
→ set VM1 node.startup=automatic
→ iSCSI login
→ restore the ext4 UUID fstab entry
→ mount and verify rw
→ restart Docker
```

On Build B:

```bash
sudo systemctl disable --now nfs-server.service
sudo exportfs -ua
sudo umount /srv/storage-b
sudo systemctl enable --now rtslib-fb-targetctl.service
sudo ss -lntp | grep ':3260'
```

Inside VM1:

```bash
sudo iscsiadm -m node \
  -T iqn.2026-08.arpa.home:storage-b.media \
  -p 192.168.1.51:3260 \
  --op update \
  -n node.startup \
  -v automatic

sudo iscsiadm -m node \
  -T iqn.2026-08.arpa.home:storage-b.media \
  -p 192.168.1.51:3260 \
  --login
```

Restore the original ext4 fstab entry, mount by UUID, confirm `rw`, and only then restart applications.

## Decision checkpoint

Keep iSCSI when it remains stable, recovery testing passes, and its block semantics are intentionally required. Migrate to NFS when simpler restart behavior and server-owned filesystem recovery are more valuable than exposing a raw LUN.

TrueNAS is not part of this data path. Routing Build B through the TrueNAS VM would add another protocol, VM, and failure point without adding redundancy.

## References

- [Frigate recording behavior](https://docs.frigate.video/configuration/record/)
- [Frigate network-storage guidance](https://docs.frigate.video/guides/ha_network_storage/)
- [Linux NFS client documentation](https://www.kernel.org/doc/html/latest/admin-guide/nfs/nfs-client.html)

