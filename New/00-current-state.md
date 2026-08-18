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
| Additional disk | 1 TB-class Samsung SATA SSD, 931.5 GiB, NTFS, intentionally untouched |
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

The 1 TB SATA SSD is deferred. It is not necessary to reformat an entire disk when adding future VMs: Proxmox allocates virtual disks from free space in a storage pool. Existing partitions only need to be changed when deliberately converting that physical disk into Proxmox storage.

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

## Address plan

| Role | Hostname | Address | Status |
|---|---|---:|---|
| Proxmox | `pve-a` | `192.168.1.10` | Active |
| VM1 services | `services-vm` | `192.168.1.20` | Active |
| VM2 drive | `drive-vm` | `192.168.1.30` | Deferred |
| VM3 owner | `owner-vm` | `192.168.1.40` | Deferred |
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
- **Tailnet DNS and exit-node routing are independent.** Connected clients use Pi-hole DNS even without selecting an exit node; ordinary internet traffic traverses VM1 only when `services-vm` is explicitly selected as the exit node.
- **DNS availability is preferred over strict enforcement.** Tailscale advertises VM1 Pi-hole plus public resolver `1.1.1.1`; clients may use either because resolver lists are not guaranteed primary/fallback ordering.
- **VM3 remains deferred.** Its owner will use a separate Tailscale organization. Host-owner console access policy remains undecided.

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
- Future private web frontends use the same DNS wildcard and certificate; only their Caddy matcher and `edge` membership change.

Pi-hole and Tailscale routing state:

- Pi-hole runs in `/home/services-vm/services/pihole` with persistent configuration at `./etc-pihole`.
- DNS listens directly on VM1 TCP/UDP port `53`; Pi-hole uses `network_mode: host` so query logs retain individual Tailscale client `100.x` addresses.
- Pi-hole's HTTP interface listens on host port `8082`; Caddy proxies `pihole.example.com` to `host.docker.internal:8082`.
- Pi-hole does not join `edge`; Caddy's Compose definition maps `host.docker.internal` using `host-gateway`.
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

## Deferred work

1. Confirm long-term Intel NIC stability; update Lenovo BIOS and Proxmox kernel during a maintenance window.
2. Decide whether iGPU passthrough is worth losing the Proxmox host’s local graphics console.
3. Migrate Vaultwarden and Jellyfin; keep every database local.
4. Complete a full Backrest plan run and test-restore representative Immich, Vaultwarden, and Personal data.
5. Create VM2 and deploy SFTPGo + Cloudflare Tunnel + Cloudflare Access.
6. Decide and create VM3 only after ownership/admin boundaries are agreed.
7. Periodically run repository checks while the external HDD is attached; plans remain manual because the disk is normally off.
8. Periodically test Pi-hole failure behavior and confirm clients can resolve through the configured public secondary resolver.
