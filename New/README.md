# Home Lab Build Wiki

**Status:** VM1's platform phase is complete: Build B storage, Proxmox, iSCSI, Frigate, Immich, shared Caddy routing, Pi-hole DNS, optional exit-node routing, monitoring design, and removable Backrest workflow are documented.  
**Last updated:** 2026-08-19  
**Scope:** Documents the completed VM1 stage, including split local/iSCSI persistence, public and private ingress, reusable Docker networking, DNS filtering, monitoring, and the normally-off USB backup repository. VM2/SFTPGo, Jellyfin, VM3, and GPU passthrough remain future work.

> **Sanitized public edition:** this is the authoritative current wiki, but it is not a literal export of the live configuration. Descriptive placeholders replace private infrastructure values. See [Placeholder reference](#placeholder-reference) before using any command.

## Current topology

```text
Internet/LAN router 192.168.1.254
        |
        +-- pve-a 192.168.1.10
        |     Proxmox VE 9.2 on 500 GB NVMe
        |     |
        |     +-- services-vm 192.168.1.20
        |           Debian 13, Docker, Frigate, Immich, Pi-hole, Caddy, cloudflared
        |           iSCSI initiator
        |
        +-- storage-b 192.168.1.51
              Debian 13 on 120 GB SSD
              LIO/targetcli iSCSI target
              Existing 1 TB HDD exported to VM1
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
| [18-netdata-monitoring.md](18-netdata-monitoring.md) | Host-network Netdata deployment, persistence, private Caddy routing, and Linux memory interpretation |
| [19-backrest-split-storage-backup.md](19-backrest-split-storage-backup.md) | Existing Restic repository, split VM1/iSCSI sources, Immich database rule, exclusions, and deduplication |
| [20-removable-backup-hdd-runbook.md](20-removable-backup-hdd-runbook.md) | Proxmox USB passthrough, UUID mount policy, automated backup sessions, safe removal, and filesystem recovery |
| [21-vm1-completion-checkpoint.md](21-vm1-completion-checkpoint.md) | Final VM1 topology, service checkpoint, invariants, direct-port behavior, health checks, and remaining phases |
| [backup-usb.sh](backup-usb.sh) | Fail-closed helper for detecting, mounting, validating, starting Backrest, stopping, syncing, and unmounting the USB repository |

## Non-negotiable safety rules

1. **Never mount the exported ext4 filesystem on Build B while VM1 has an iSCSI session.** ext4 is not a clustered filesystem; two writers can corrupt it.
2. **Never run `fsck` against an online LUN.** Shut down VM1, confirm no sessions, clear the live LIO configuration, then repair.
3. **Never identify the LUN by `/dev/sdb` or `/dev/sdc`.** Those names are dynamic. Use filesystem UUID and HDD PARTUUID.
4. **Keep databases local to VM1.** Frigate SQLite, Immich PostgreSQL, and Vaultwarden SQLite must not live on iSCSI.
5. **Keep the external restic/Backrest disk powered off except during backup operations.** Snapshots and RAID are not substitutes for that offline copy.
6. **Never expose administrative services through an unintended exact DNS record.** Private hostnames must resolve through the DNS-only wildcard to VM1's Tailscale IP; public tunnel CNAMEs are explicit exceptions.
7. **Attach only web frontends to Docker network `edge`.** Databases and caches stay on application-private networks.
8. **Treat Pi-hole as the documented host-network exception.** Host networking preserves client IPs; its web UI uses host port `8082` and Caddy reaches it through `host.docker.internal`.
9. **Mount the removable repository before recreating Backrest.** A container started against the empty mountpoint can mistake it for a new repository.
10. **Disconnect the backup HDD only after `backup-usb stop` reports `SAFE TO REMOVE`.** An idle UI does not prove Restic has stopped or writes are flushed.

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

Real domains, Tailscale `100.x` address
- Hardware serial numbers and product UUIDs

Replace every value shown as `<SECRET>` or `<PASSWORD>` with the value stored in the password manager.
