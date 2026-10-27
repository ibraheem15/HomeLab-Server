# 00 — Current State

Last confirmed: 2026-08-25. Verify live state before changing it; this file records intended and last-known state, not a substitute for host checks.

## Inventory

| System | Role | LAN / storage IP | Persistent storage | State |
|---|---|---|---|---|
| `pve-a` | Proxmox VE 9.2; Lenovo M70q, i7-10700T, 48 GiB | `192.168.1.10` | 500 GB NVMe; ~50 GB VG free | Active |
| TrueNAS VM `101` | ZFS/SMB | `192.168.1.25` / `10.20.0.10` | whole 1 TB Samsung 870 EVO → pool `sata` | Active |
| `services-vm` VM `102` | Docker, Caddy, cloudflared, Frigate, Immich, AdGuard | `192.168.1.20` | 100 GB local NVMe + Build B iSCSI at `/mnt/storage-b` | Active |
| `baadesaba` VM `103` | SFTPGo | `192.168.1.30` / `10.20.0.20` | 20 GB local + `//10.20.0.10/vm2-business` | Active |
| `owner-vm` VM `104` | future owner workload | `192.168.1.40` / `10.20.0.30` | 30 GB local + planned `//10.20.0.10/vm3-extra` | Debian installed; configuration pending |
| `storage-b` | Debian 13 LIO iSCSI target | `192.168.1.51` | 128 GB boot SSD; preserved 1 TB HDD ext4 LUN | Active |

Gateway: `192.168.1.254`. Static addressing lives inside each OS. Tailscale is remote administration, not a storage transport.

## Storage contract

| Owner | Data | Consumer |
|---|---|---|
| Proxmox NVMe | VM OS disks and all databases/config | Proxmox / VM local disks |
| TrueNAS `sata` | `vm2-business` (100 GiB quota), `vm3-extra` (500 GiB quota) | VM2/VM3 through SMB over `vmbr1` |
| Build B HDD | Frigate media, Immich assets | VM1 only through iSCSI |
| External HDD | Restic/Backrest repository | attached only for backup sessions |

The `sata` pool is single-disk ZFS: no redundancy. Keep 10–20% free. Build B’s HDD contains a retained old Debian tree; do not format it during unrelated work.

## Working services

- VM1: Frigate database/config local; media on `/mnt/storage-b`. Immich PostgreSQL local at `/home/services-vm/services/immich/postgres`; asset library remains on Build B. Caddy and `cloudflared` are under `/home/services-vm/services/proxy`.
- Caddy listens only on `127.0.0.1:8443`; persistent Tailscale Serve owns tailnet TCP `:443`. `edge` is the shared external Docker network.
- Cloudflare’s active VM1 tunnel is `services-vm`. The exact public `camera` record is proxied; the private wildcard is DNS-only to VM1’s Tailscale address.
- AdGuard Home uses host networking, owns TCP/UDP `53`, and exposes its UI on `8083`. Pi-hole and Technitium are stopped rollback options.
- VM2 keeps SFTPGo SQLite, config, and host keys under `/home/baadesaba/services/sftpgo/state`; uploads live on the SMB mount. Its Cloudflare connector exposes web UI only.

## Decisions and unfinished work

- ACL + CHAP protect Build B; VM1 is the sole iSCSI initiator. Its target IQN and UUID/PARTUUID are in the password manager and component guides.
- GPU passthrough, Jellyfin, Vaultwarden migration, SFTPGo 2FA, VM3 networking/storage/Tailscale, and the iSCSI→NFS migration are not complete.
- Perform a planned full-reboot test for AdGuard DNS and private Caddy/Tailscale Serve persistence; do not claim it has been verified.
- Monitor the SATA SSD’s reported reallocated sectors and maintain tested offline backups.
