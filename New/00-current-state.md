# 00 — Current State and Architecture

## Hardware discovered during the build

### Build A — `pve-a`

| Component | Actual state |
|---|---|
| Model | Lenovo ThinkCentre M70q |
| CPU | Intel Core i7-10700T, 8 cores / 16 threads |
| RAM | 48 GB |
| GPU | Intel UHD Graphics 630 at PCI `00:02.0`, not passed through |
| Boot disk | 500 GB-class Samsung NVMe, 476.9 GiB usable |
| Additional disk | Samsung 870 EVO 1 TB SATA SSD, 931.5 GiB, passed through to TrueNAS VM `101` |
| Hypervisor | Proxmox VE 9.2 on Debian 13 base |
| Address | `192.168.1.10/24`; gateway `192.168.1.254` |

Proxmox NVMe layout:

```text
nvme0n1                         476.9G
├─nvme0n1p2  /boot/efi            1G
└─nvme0n1p3  LVM PV             475.9G
  ├─pve-swap                       8G
  ├─pve-root                      64G
  ├─pve-data thin pool          346.9G
  └─VG free                      50.0G
```

The former NTFS partition was intentionally erased after explicit confirmation. The entire SATA SSD is now a single-disk ZFS pool named `sata`, owned exclusively by TrueNAS. Proxmox must not mount it or add it as ordinary Proxmox storage. SMART passed, but the SSD reported 34 reallocated sectors; it requires monitoring and is not a substitute for backup.

### Build B — `storage-b`

| Component | Actual state |
|---|---|
| Model | Dell OptiPlex 7040 |
| RAM | 8 GB |
| Boot disk | Lexar 128 GB SSD, 119.2 GiB usable |
| Data disk | WDC WD10SPZX, 1 TB-class, 931.5 GiB usable |
| OS | Debian 13 `trixie` |
| Address | `192.168.1.51/24`; gateway `192.168.1.254` |
| Function | Storage-only iSCSI target |

The project initially described the HDD as 1.5 TB. Hardware inspection established that the installed disk is actually 1 TB nominal / 931.5 GiB.

Boot SSD layout:

```text
sda                         119.2G  Lexar 128GB SSD
├─sda1  ext4  /             113.2G
└─sda5  swap                  6.1G
```

Preserved HDD layout:

```text
sdb                         931.5G  WDC WD10SPZX
├─sdb1  vfat                 512M   old EFI partition
├─sdb2  ext4               930.1G   old Debian root + all retained data
└─sdb3  swap                 977M   old swap partition
```

Stable identifiers:

```text
Filesystem UUID: 675a9a2e-<SECRET>-ff0a0c4ec61e
Partition UUID:  b08c7158-<SECRET>-f839eab06a67
```

The HDD was not wiped. The old Debian directory tree remains visible inside the exported filesystem; application media is referenced within that tree.

### VM1 — `services-vm`

| Setting | Current value |
|---|---|
| Proxmox VMID | `102` |
| OS | Debian 13 |
| Address | `192.168.1.20/24`; gateway `192.168.1.254` |
| CPU | 6 vCPU, CPU type `host` |
| RAM | 24 GiB, ballooning disabled |
| Local disk | 100 GiB on `local-lvm` |
| Firmware/machine | OVMF/UEFI + Q35 |
| Network | VirtIO on `vmbr0`, no VLAN tag |
| iSCSI role | Sole authorized initiator for Build B |

VM1 initiator IQN:

```text
iqn.1993-08.org.debian:01:7feb45<SECRET>ced7d
```

VM1 local disk contains Debian, Docker, Compose definitions, configuration, and databases. The iSCSI LUN mounts at `/mnt/storage-b`; `/home/services-vm/storage-b` is a persistent symlink for convenience.

### TrueNAS — VM `101`

| Setting | Current value |
|---|---|
| OS | TrueNAS Community Edition 25.10.6 |
| VMID | `101` |
| Management | `192.168.1.25/24` on `vmbr0` |
| Storage network | `10.20.0.10/24` on `vmbr1`; no gateway |
| CPU/RAM | 2 host-type vCPU / 8 GiB fixed RAM |
| Boot disk | 32 GiB local NVMe virtual disk |
| Data disk | Whole Samsung 870 EVO 1 TB SATA SSD via stable by-id path |
| Pool | `sata`, single-disk ZFS stripe, no encryption/redundancy |

Datasets:

```text
sata
├── vm2-business   quota 100 GiB, no reservation
└── vm3-extra     quota 500 GiB, no reservation
```

Quotas are adjustable limits, not preallocated partitions. They can be increased online; actual combined usage must still fit the physical pool. Keep 10–20% free.

### VM2 — `baadesaba`

| Setting | Current value |
|---|---|
| VMID | `103` |
| OS | Debian 13 |
| LAN | `192.168.1.30/24`, gateway `192.168.1.254` |
| Storage NIC | `10.20.0.20/24`, no gateway |
| CPU/RAM | 4 host-type vCPU / 4 GiB ballooned to 2 GiB minimum |
| Local disk | 20 GiB on `local-lvm` |
| Shared data | `//10.20.0.10/vm2-business` at `/mnt/vm2-business` |
| Services | Docker, Tailscale, SFTPGo, dedicated `cloudflared` connector |

SFTPGo state/SQLite/host keys stay under `/home/baadesaba/services/sftpgo/state` on local NVMe. Uploaded files live under `/mnt/vm2-business/baadesaba` on TrueNAS.

### VM3

| Setting | Current value |
|---|---|
| VMID | `104` |
| OS | Debian 13 installed |
| CPU/RAM | 4 host-type vCPU / 4 GiB ballooned to 2 GiB minimum |
| Local disk | 30 GiB on `local-lvm` |
| Target LAN | `192.168.1.40/24` |
| Target storage NIC | `10.20.0.30/24` |
| Target share | `//10.20.0.10/vm3-extra` |
| Status | Networking, SMB mount, Tailscale organization, and workload configuration pending |

## Address plan

| Role | Hostname | Address | Status |
|---|---|---:|---|
| Proxmox | `pve-a` | `192.168.1.10` | Active |
| VM1 services | `services-vm` | `192.168.1.20` | Active |
| TrueNAS management | `truenas` | `192.168.1.25` | Active |
| VM2 drive | `baadesaba` | `192.168.1.30` | Active |
| VM3 | user-selected | `192.168.1.40` | Debian installed; network pending |
| Build B storage | `storage-b` | `192.168.1.51` | Active; `.51` chosen instead of original `.50` plan |
| Gateway/router | — | `192.168.1.254` | Existing LAN gateway |

## Locked architecture decisions

- **VM1 is the iSCSI initiator.** Proxmox does not install or log into `open-iscsi`; this enforces that only VM1 sees the data LUN.
- **ACL + CHAP are both required.** IQN ACL restricts identity; CHAP supplies an actual credential.
- **Databases remain on VM1 local storage.** Network loss can return block-level I/O errors and abort a journal, as observed during the NIC incident.
- **Build B stays Debian + ext4.** ZFS was evaluated but not adopted because the existing filesystem had to be retained without wiping and Build B has only 8 GB RAM.
- **No RAID1.** Capacity is limited and no redundant matching drive exists.
- **No router changes.** Static addressing is configured on each OS.
- **Tailscale is an administration path, not the storage path.** iSCSI uses LAN addresses. Private custom-domain services resolve to VM1's Tailscale IP and are unreachable without tailnet connectivity.
- **One Cloudflare Tunnel serves public VM1 applications.** The active tunnel is `services-vm`; application names such as `camera.example.com` are routes, not separate tunnels.
- **Caddy is the shared reverse proxy.** `cloudflared` reaches Caddy on private Docker network `edge`; future web frontends join `edge`, while databases remain isolated.
- **Public and private DNS are intentionally different.** Exact proxied tunnel CNAMEs are public; the DNS-only wildcard resolves undefined subdomains to VM1's Tailscale IP.
- **Tailnet DNS and exit-node routing are independent.** Connected clients use AdGuard Home DNS even without selecting an exit node; ordinary internet traffic traverses VM1 only when `services-vm` is explicitly selected as the exit node.
- **DNS availability is preferred over strict enforcement.** Tailscale advertises VM1 AdGuard Home plus public resolver `1.1.1.1`; clients may use either because resolver lists are not guaranteed primary/fallback ordering.
- **TrueNAS owns the whole SATA SSD.** VM2 and VM3 consume SMB datasets through private bridge `vmbr1`; Proxmox does not mount the ZFS disk.
- **Dataset quotas replace fixed partitions.** `vm2-business` starts at 100 GiB and `vm3-extra` at 500 GiB; both remain expandable without reformatting.
- **Build B HDD remains the VM1 media tier.** Immich media and Frigate recordings stay on HDD/iSCSI; PostgreSQL stays on VM1 NVMe.
- **VM3 is installed but not operational.** It will eventually join a separate Tailscale organization; host-owner console policy remains undecided.

## Current service state

Frigate is running in VM1:

- Compose and `/config` reside on VM1’s 100 GiB local disk.
- Recordings, clips, and snapshots bind-mount from Build B’s iSCSI filesystem.
- OpenVINO detector uses CPU.
- Intel GPU passthrough is not configured.
- Authenticated Frigate UI uses port `8971`.
- Internal unauthenticated port `5000` and go2rtc administration port `1984` are not published.
- `camera.example.com` is working through Cloudflare Tunnel -> Caddy -> Frigate `8971`.
- Frigate internal TLS is disabled because Cloudflare terminates public HTTPS and Caddy uses private HTTP to port `8971`.

Immich is running in VM1:

- Deployment directory: `/home/services-vm/services/immich`.
- PostgreSQL directory: `/home/services-vm/services/immich/postgres` on VM1's local SSD.
- Existing 89 GiB asset root remains at `/mnt/storage-b/home/<LEGACY_USER>/immich-app/library`.
- Restore source: `immich-db-backup-<TIMESTAMP>-v3.0.3-pg14.19.sql.gz`.
- Initial restored version is pinned to `v3.0.3` to match the backup.
- `immich-server` joins `immich-internal` and `edge`; PostgreSQL, Valkey, and ML join only `immich-internal`.
- Private Caddy upstream: `immich-server:2283`.

Reverse-proxy state:

- `cloudflared` and Caddy run in `/home/services-vm/services/proxy`.
- Active tunnel: `services-vm`; an accidentally created inactive `camera` tunnel was removed.
- Docker external network: `edge`.
- Exact `camera` DNS record points to the active tunnel and is proxied.
- DNS-only wildcard `*` points at VM1's Tailscale IPv4 for future private services.
- Portainer joins `default`/`portainer_network` plus `edge`.
- Caddy wildcard DNS-01 support uses official Caddy images plus `caddy-dns/cloudflare` source.
- Caddy publishes container TLS only to host loopback `127.0.0.1:8443`; it no longer binds Docker directly to the Tailscale `100.x` address.
- Persistent Tailscale Serve raw TCP forwarding owns tailnet port `443` and forwards to `127.0.0.1:8443`, preserving Caddy TLS termination while removing the Tailscale-address boot race.
- The new private HTTPS path was verified working on 2026-08-25. Full VM reboot persistence remains pending an approved maintenance window.
- Future private web frontends use the same DNS wildcard and certificate; only their Caddy matcher and `edge` membership change.

AdGuard Home and Tailscale routing state:

- AdGuard Home runs in `/home/services-vm/services/adguardhome`; `./conf` and `./work` persist on VM1's local NVMe-backed disk.
- AdGuard Home uses `network_mode: host`, owns VM1 TCP/UDP port `53`, and retains individual LAN/Tailscale client addresses instead of a Docker `172.x` gateway.
- Its HTTP interface listens on host port `8083`; private Caddy routing proxies `adguard.example.com` to `host.docker.internal:8083`.
- AdGuard Home does not join `edge`; Caddy's Compose definition maps `host.docker.internal` using `host-gateway`.
- DNS over UDP and TCP, filtering, cache behavior, the private HTTPS route, and distinct Tailscale `100.x` client attribution were verified on 2026-08-24. A full VM1 reboot/persistence test remains pending a maintenance window.
- Pi-hole remains installed at `/home/services-vm/services/pihole`, with `./etc-pihole` and a Teleporter export preserved. Its container is stopped with restart policy `no`.
- A Technitium trial remains at `/home/services-vm/services/technitium`; its container is stopped with restart policy `no`. It is not an active resolver.
- VM1 uses `tailscale set --accept-dns=false` to avoid resolving through itself.
- VM1 advertises an exit node; clients opt in individually. DNS filtering does not imply full-traffic routing.
- Tailscale global nameservers are VM1's stable Tailscale IPv4 plus `1.1.1.1`, with Override DNS enabled and MagicDNS retained.

Monitoring and backup state:

- Netdata's documented layout uses host networking on port `19999`; it cannot simultaneously join Docker network `edge`.
- Caddy reaches host-network services through `host.docker.internal`, backed by the `host-gateway` mapping.
- Backrest Compose resides at `/home/services-vm/services/backrest`.
- The existing Restic repository is on a 500 GB-class USB HDD with ext4 UUID `ea3f8b0d-<SECRET>-e4029de67a34`.
- Proxmox passes the USB device from Build A into VM1; VM1 mounts it manually at `/mnt/backup-seagate`.
- The repository root contains `config`, `data`, `index`, `keys`, `locks`, `snapshots`, and the `is_mounted` sentinel.
- Backrest backs up Personal data, Immich local configuration, Immich iSCSI assets/SQL dumps, and active local Vaultwarden data. It excludes Immich's live PostgreSQL directory and the obsolete Build B Vaultwarden duplicate.
- `/usr/local/sbin/backup-usb` automates detection, mount validation, Backrest recreation, active-Restic refusal, sync, unmount, and safe-removal confirmation.
- The USB repository remains powered off and unmounted outside manual backup sessions.

VM2/SFTPGo state:

- TrueNAS SMB account `vm2_business` owns `sata/vm2-business`; its Unix home remains `/var/empty` because SMB access is ACL-based.
- VM2 mounts the share using a root-only credentials file and systemd CIFS automount.
- SFTPGo runs as UID/GID 1000, matching the CIFS client mapping.
- `cloudflared` runs beside SFTPGo on Compose network `sftpgo_default` and routes the public drive hostname to `http://sftpgo:8080`.
- Cloudflare Access was intentionally skipped. SFTPGo credentials are the current public protection layer; 2FA and brute-force hardening remain deferred.

## Deferred work

1. Confirm long-term Intel NIC stability; update Lenovo BIOS and Proxmox kernel during a maintenance window.
2. Decide whether iGPU passthrough is worth losing the Proxmox host’s local graphics console.
3. Migrate Vaultwarden and Jellyfin; keep every database local.
4. Complete a full Backrest plan run and test-restore representative Immich, Vaultwarden, and Personal data.
5. Enable SFTPGo admin/user 2FA and review brute-force/rate-limit settings before broader account distribution.
6. Configure VM3 static LAN/storage networking, persistent SMB mount, guest agent, and its separate Tailscale organization.
7. Periodically run repository checks while the external HDD is attached; plans remain manual because the disk is normally off.
8. Reboot-test AdGuard Home persistence during a maintenance window, then periodically test failure behavior and public-secondary resolution.
9. Add `/home/services-vm/services/adguardhome/{conf,work}` to the next appropriate Backrest plan and test a representative configuration restore.
