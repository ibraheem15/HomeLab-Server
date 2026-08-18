# Home Lab Build Wiki

**Status:** Build B storage, Proxmox, VM1 iSCSI, Frigate, Immich restoration, shared Caddy routing, Pi-hole tailnet DNS, and the optional VM1 exit node are operational.  
**Last updated:** 2026-08-18  
**Scope:** Documents the build through Frigate and Immich migration, public `camera.example.com`, private wildcard HTTPS, reusable Docker/Caddy networking, Pi-hole filtering, and Tailscale exit-node routing. VM2/SFTPGo, Jellyfin, Vaultwarden, VM3, and GPU passthrough remain future work.

> **Sanitized public edition:** this is the authoritative current wiki, but it is not a literal export of the live configuration. Descriptive placeholders replace private infrastructure values. See [Placeholder reference](#placeholder-reference) before using any command.

## Current topology

```text
Router / switch <LAN_GATEWAY_IP>
├── pve-a <PVE_LAN_IP>
│   ├── Proxmox VE 9.2 on 500 GB NVMe
│   └── services-vm <SERVICES_VM_LAN_IP>
│       ├── Debian 13 and Docker
│       ├── Frigate, Immich, Pi-hole, Caddy, and cloudflared
│       └── sole iSCSI initiator
└── storage-b <STORAGE_SERVER_LAN_IP>
    ├── Debian 13 on 120 GB SSD
    ├── LIO/targetcli iSCSI target
    └── existing 1 TB HDD exported to services-vm
```

All static addresses are configured in the operating systems. No DHCP reservations or port forwarding are required. Tailscale is installed on Build A, Build B, and VM1. Public Frigate access uses an outbound Cloudflare Tunnel; private custom-domain services resolve to VM1's Tailscale address through a DNS-only wildcard.

## Wiki map

| File | Purpose |
|---|---|
| [00-current-state.md](00-current-state.md) | Authoritative inventory, addresses, storage layout, decisions, and unfinished work |
| [01-preparation-and-network.md](01-preparation-and-network.md) | Downloads, physical preparation, IP plan, Debian USB, and network troubleshooting |
| [02-build-b-storage-server.md](02-build-b-storage-server.md) | Install Debian on the 120 GB SSD while preserving the original HDD |
| [03-build-b-iscsi-target.md](03-build-b-iscsi-target.md) | Export the existing ext4 partition with targetcli, ACL, CHAP, and persistence |
| [04-build-a-proxmox.md](04-build-a-proxmox.md) | Proxmox installation, actual disk layout, Tailscale, and storage policy |
| [05-vm1-services-vm.md](05-vm1-services-vm.md) | VM1 creation, Debian networking, template accident, and guest preparation |
| [06-vm1-iscsi-initiator.md](06-vm1-iscsi-initiator.md) | CHAP login, persistent mount, dynamic device names, and mount safety |
| [07-incident-intel-nic-hang.md](07-incident-intel-nic-hang.md) | `e1000e` hardware-unit hang: diagnosis, recovery, and persistent prevention |
| [08-incident-iscsi-ext4-recovery.md](08-incident-iscsi-ext4-recovery.md) | iSCSI timeout, aborted ext4 journal, SMART interpretation, and offline repair |
| [09-frigate-migration.md](09-frigate-migration.md) | Local database vs. remote media layout, corrected Compose file, and config migration |
| [10-operations-runbook.md](10-operations-runbook.md) | Reboot order, health checks, backup rules, outage handling, and routine maintenance |
| [11-troubleshooting-index.md](11-troubleshooting-index.md) | Symptom → cause → corrective action quick reference |
| [12-cloudflare-tunnel-and-caddy.md](12-cloudflare-tunnel-and-caddy.md) | Namecheap/Cloudflare DNS delegation, tunnel deployment, Caddy routing, Frigate TLS, and duplicate-tunnel cleanup |
| [13-tailscale-private-domains.md](13-tailscale-private-domains.md) | Wildcard DNS to VM1's Tailscale IP, private/public hostname policy, and why DNS-01 is required |
| [14-caddy-wildcard-tls-and-edge-network.md](14-caddy-wildcard-tls-and-edge-network.md) | Scoped API token, official-source Caddy build, wildcard certificate, shared Docker network, and SSL troubleshooting |
| [15-immich-migration.md](15-immich-migration.md) | Version-matched restore, local PostgreSQL, preserved iSCSI assets, permission/DNS incidents, and Caddy integration |
| [16-reusable-docker-edge-network.md](16-reusable-docker-edge-network.md) | Standard frontend/internal network topology and repeatable procedure for adding future services |
| [17-pihole-tailscale-dns-and-exit-node.md](17-pihole-tailscale-dns-and-exit-node.md) | Tailnet-wide DNS filtering, public fallback resolver, optional exit-node routing, host-network deployment, and incident fixes |

## Non-negotiable safety rules

1. **Never mount the exported ext4 filesystem on Build B while VM1 has an iSCSI session.** ext4 is not a clustered filesystem; two writers can corrupt it.
2. **Never run `fsck` against an online LUN.** Shut down VM1, confirm no sessions, clear the live LIO configuration, then repair.
3. **Never identify the LUN by `/dev/sdb` or `/dev/sdc`.** Those names are dynamic. Use filesystem UUID and HDD PARTUUID.
4. **Keep databases local to VM1.** Frigate SQLite, Immich PostgreSQL, and Vaultwarden SQLite must not live on iSCSI.
5. **Keep the external restic/Backrest disk powered off except during backup operations.** Snapshots and RAID are not substitutes for that offline copy.
6. **Never expose administrative services through an unintended exact DNS record.** Private hostnames must resolve through the DNS-only wildcard to VM1's Tailscale IP; public tunnel CNAMEs are explicit exceptions.
7. **Attach only web frontends to Docker network `edge`.** Databases and caches stay on application-private networks.
8. **Treat Pi-hole as the documented host-network exception.** Host networking preserves client IPs; its web UI uses host port `8082` and Caddy reaches it through `host.docker.internal`.

## Placeholder reference

The public wiki intentionally omits all live identifiers, not only passwords.

| Placeholder | Meaning |
|---|---|
| `<LAN_CIDR>` / `<LAN_GATEWAY_IP>` | Private LAN and router address |
| `<PVE_LAN_IP>` | Proxmox host address |
| `<SERVICES_VM_LAN_IP>` | Docker services VM address |
| `<STORAGE_SERVER_LAN_IP>` | iSCSI target address |
| `<DRIVE_VM_LAN_IP>` / `<OWNER_VM_LAN_IP>` | Reserved future guest addresses |
| `<CAMERA_1_LAN_IP>` / `<CAMERA_2_LAN_IP>` | Camera addresses |
| `<SERVICES_VM_TAILSCALE_IP>` | Services VM address inside the tailnet |
| `<FILESYSTEM_UUID>` / `<PARTITION_UUID>` | Stable identifiers for the exported filesystem |
| `<INITIATOR_IQN>` / `<TARGET_IQN>` | iSCSI identities |
| `<CHAP_USERNAME>` | Non-secret iSCSI login name, omitted to avoid publishing a complete login profile |
| `<LEGACY_USER>` | Username retained in paths on the migrated filesystem |
| `<IANA_TIMEZONE>` | Deployment timezone, for example `Etc/UTC` |
| `<TIMESTAMP>` | Redacted date/time embedded in a backup filename |
| `<SECRET>` and related names | Value held outside Git in a password manager or protected environment file |

Real domains, Tailscale `100.x` addresses, tunnel UUIDs, hardware serial numbers, product UUIDs, personal usernames, and backup timestamps are also omitted. Generic service-role names remain. Never paste a placeholder literally into a live configuration.
