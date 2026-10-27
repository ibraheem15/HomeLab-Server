# Home Lab Wiki

Operational notes for rebuilding and running the lab. `00-current-state.md` is the source of truth; numbered files record build and recovery procedures. Secrets belong in the password manager, never here.

## Architecture

```mermaid
flowchart TB
  classDef external fill:#eaf2ff,stroke:#2563eb,color:#172554
  classDef network fill:#ecfdf5,stroke:#059669,color:#064e3b
  classDef compute fill:#fff7ed,stroke:#ea580c,color:#7c2d12
  classDef storage fill:#f5f3ff,stroke:#7c3aed,color:#4c1d95
  classDef service fill:#fefce8,stroke:#ca8a04,color:#713f12
  Internet[Internet]:::external
  Tailnet[Tailscale clients]:::external
  Router[LAN router<br/>192.168.1.254]:::network
  Cloudflare[Cloudflare<br/>public DNS + Tunnel]:::external
  subgraph LAN["Home LAN · 192.168.1.0/24"]
    direction TB
    Router
    subgraph PVE["pve-a · 192.168.1.10 · Proxmox VE"]
      direction LR
      subgraph VM1["services-vm · VM 102 · 192.168.1.20"]
        direction TB
        TS[Tailscale + Serve :443]:::service
        Tunnel[cloudflared]:::service
        Caddy[Caddy · edge network<br/>loopback TLS :8443]:::service
        AdGuard[AdGuard Home<br/>host network :53 / :8083]:::service
        Frigate[Frigate<br/>config + SQLite on local NVMe]:::service
        Immich[Immich<br/>PostgreSQL on local NVMe]:::service
      end
      subgraph NAS["TrueNAS · VM 101"]
        TrueNAS[LAN 192.168.1.25<br/>storage 10.20.0.10<br/>ZFS pool sata]:::compute
      end
      VM2[baadesaba · VM 103<br/>192.168.1.30 / 10.20.0.20<br/>SFTPGo]:::compute
      VM3[owner-vm · VM 104<br/>192.168.1.40 / 10.20.0.30<br/>pending configuration]:::compute
    end
    StorageB[storage-b · 192.168.1.51<br/>Debian LIO iSCSI target]:::compute
  end
  NVMe[Proxmox NVMe<br/>VM disks + VM1 app DBs/config]:::storage
  SATA[1 TB SATA SSD<br/>passed through to TrueNAS]:::storage
  HDD[1 TB HDD ext4<br/>actual Frigate + Immich data]:::storage
  Backup[External Restic HDD<br/>normally disconnected]:::storage
  Internet --> Cloudflare
  Cloudflare -->|public HTTP| Tunnel
  Tailnet -->|private service access| TS
  TS -->|tailnet :443| Caddy
  Tunnel --> Caddy
  Caddy --> Frigate
  Caddy --> Immich
  Tailnet -->|DNS| AdGuard
  Router --> PVE
  Router --> StorageB
  SATA -->|whole disk passthrough| TrueNAS
  TrueNAS -->|SMB vm2-business| VM2
  TrueNAS -->|SMB vm3-extra| VM3
  StorageB -->|iSCSI; VM1 only| Frigate
  StorageB -->|iSCSI; VM1 only| Immich
  NVMe -. local state .-> Frigate
  NVMe -. local DB .-> Immich
  Backup -. backup session .-> NVMe
  Backup -. backup session .-> HDD
  HDD --- StorageB
```

Public HTTP uses Cloudflare Tunnel → Caddy; Tailscale reaches `services-vm` directly for DNS, administration, and private HTTPS through Serve → Caddy. VM1 local NVMe owns Frigate configuration/SQLite and Immich PostgreSQL; the Build B HDD holds their actual media data through VM1’s exclusive iSCSI mount. Native SFTP uses VM2’s LAN/Tailscale address on port 2022.

## Read order

1. [00-current-state.md](00-current-state.md) — live inventory and invariants.
2. [10-operations-runbook.md](10-operations-runbook.md), then [11-troubleshooting-index.md](11-troubleshooting-index.md).
3. Build docs `01`–`08`, service docs `09`, `12`–`20`, then TrueNAS/VM docs `22`–`27`.
4. `28-build-b-outage-recovery.md` handles an outage; `28-future-iscsi-to-nfs-migration.md` is deferred.

## Non-negotiable rules

- Identify disks by `/dev/disk/by-id`, UUID, or PARTUUID; never by `sdX`.
- Build B ext4 is mounted through VM1 iSCSI or locally on Build B, never both.
- TrueNAS exclusively owns the passed-through SATA SSD and exports child datasets, not the pool root.
- Storage-dependent containers fail closed when their mount is absent.
- Only web frontends join Docker network `edge`; databases and caches stay private.
- Explicit Cloudflare routes are public; wildcard private DNS points to VM1’s Tailscale address.
