# Home Lab Infrastructure Wiki

This repository documents the design, migration, recovery, and day-to-day operation of a two-node home lab built around Proxmox, Debian, Docker, iSCSI, Tailscale, Caddy, Cloudflare Tunnel, Frigate, Immich, and Pi-hole.

The documentation is written as an operational wiki rather than a collection of isolated installation notes. It records architecture decisions, failure modes, recovery procedures, and reusable deployment patterns.

> **Public-repository notice:** all live addresses, storage identifiers, initiator identifiers, personal account paths, domains, and credentials are replaced with descriptive placeholders. Role names such as `services-vm` remain for architectural clarity. Values such as `<STORAGE_SERVER_LAN_IP>` are not meant to be pasted literally.

## Architecture at a glance

```text
Router / switch
├── Build A: Proxmox VE
│   └── Services VM: Docker workloads, local databases, reverse proxy, DNS
└── Build B: Debian storage server
    └── Existing ext4 filesystem exported exclusively to the Services VM by iSCSI

Remote/private access: Tailscale
Selected public access: Cloudflare Tunnel -> Caddy -> application frontend
Bulk application media: iSCSI filesystem
Application databases: Services VM local SSD
Offline recovery copy: restic/Backrest repository
```

## Documentation

| Area | Status | Entry point |
|---|---|---|
| Current build wiki | Authoritative | [New/README.md](New/README.md) |
| Current architecture and inventory | Authoritative | [New/00-current-state.md](New/00-current-state.md) |
| Operations and recovery | Authoritative | [New/10-operations-runbook.md](New/10-operations-runbook.md) |
| Troubleshooting index | Authoritative | [New/11-troubleshooting-index.md](New/11-troubleshooting-index.md) |
| Previous-generation notes | Archived; may be obsolete | [Old/README.md](Old/README.md) |
| Disclosure and secret-handling policy | Current | [SECURITY.md](SECURITY.md) |

## Engineering themes

- Single-writer storage ownership for a non-clustered ext4 iSCSI LUN.
- Local placement for SQLite and PostgreSQL databases to contain storage-path failures.
- Stable UUID/PARTUUID-based mounts instead of dynamic `/dev/sdX` names.
- Recovery procedures built from real NIC, iSCSI, ext4, Docker DNS, and reverse-proxy incidents.
- Explicit separation between public Cloudflare Tunnel routes and Tailscale-only services.
- Shared Caddy ingress networking without attaching databases or caches to the edge network.
- Offline backups and tested restores instead of treating snapshots or RAID as backups.

## Repository status

The `New/` directory describes the current build. `Old/` is retained only to show the project’s evolution and must not be used as current deployment guidance.
