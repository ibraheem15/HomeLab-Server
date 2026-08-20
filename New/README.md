# Home Lab Build Wiki

**Status:** VM1 is complete for now. TrueNAS VM `101`, pool `sata`, isolated VM2/VM3 datasets, VM2/SFTPGo, persistent SMB storage, Tailscale, Docker, and a dedicated public Cloudflare Tunnel are now operational. VM3 has Debian installed but still needs static networking, storage mounting, Tailscale, and owner-specific configuration.

**Last updated:** 2026-08-20

**Scope:** Documents Build B, Proxmox, VM1, TrueNAS, VM2/SFTPGo, public/private ingress, DNS filtering, monitoring, split-storage backup, Build B outage recovery, and the current VM3 checkpoint. Jellyfin, Vaultwarden migration, VM3 finalization, SFTPGo 2FA, and GPU passthrough remain future work.

> **Sanitized public edition:** this is the authoritative current wiki, but it is not a literal export of the live configuration. Descriptive placeholders replace private infrastructure values. See [Placeholder reference](#placeholder-reference) before using any command.

## Current topology

```text
Internet/LAN router 192.168.1.254
        |
        +-- pve-a 192.168.1.10
        |     Proxmox VE 9.2 on 500 GB NVMe
        |     |
        |     +-- truenas 192.168.1.25 / 10.20.0.10
        |     |     1 TB SATA SSD -> ZFS pool sata
        |     |     SMB datasets vm2-business + vm3-extra
        |     |
        |     +-- services-vm 192.168.1.20
        |           Debian 13, Docker, Frigate, Immich, Pi-hole, Caddy, cloudflared
        |           iSCSI initiator
        |     |
        |     +-- VM2 baadesaba 192.168.1.30 / 10.20.0.20
        |     |     SFTPGo + dedicated cloudflared connector
        |     |
        |     +-- VM3 192.168.1.40 / 10.20.0.30 planned
        |           Debian installed; configuration pending
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
| [22-build-a-storage-architecture.md](22-build-a-storage-architecture.md) | Final NVMe/SATA/HDD roles, TrueNAS rationale, private storage bridge, and capacity model |
| [23-truenas-vm-and-zfs-pool.md](23-truenas-vm-and-zfs-pool.md) | TrueNAS VM creation, raw SSD attachment, installer/display issues, serial fix, networking, and ZFS pool creation |
| [24-truenas-datasets-acls-and-smb.md](24-truenas-datasets-acls-and-smb.md) | Dataset quotas, SMB identities, ACL isolation, Linux SMB rationale, and share definitions |
| [25-vm2-network-and-storage.md](25-vm2-network-and-storage.md) | VM2 build, dual-NIC static networking, DHCP/DNS incidents, persistent SMB mount, Tailscale, and Docker baseline |
| [26-sftpgo-and-cloudflare-tunnel.md](26-sftpgo-and-cloudflare-tunnel.md) | SFTPGo split persistence, UID fix, initial user, dedicated tunnel, published route, and deferred hardening |
| [27-vm3-installation-checkpoint.md](27-vm3-installation-checkpoint.md) | VM3 resources, installed state, target network/storage configuration, and unfinished work |
| [28-build-b-outage-recovery.md](28-build-b-outage-recovery.md) | Build B outage recovery while VM1 remains running, stale-mount escalation, and UPS shutdown order |
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
11. **Do not mount or format the 1 TB SATA SSD on Proxmox.** TrueNAS VM `101` owns the whole physical disk and ZFS pool `sata`.
12. **Do not share the root of pool `sata`.** Export only child datasets with isolated ACLs.
13. **Keep VM2/VM3 application databases on their local NVMe disks.** SMB is for bulk files, not databases.
14. **Never start file services against an unmounted SMB path.** Confirm `findmnt` before Docker recreation; systemd automount is the normal guard.

## Secrets intentionally omitted

The wiki includes real LAN addresses, target IQN, filesystem UUID, and partition UUID because they are required for reconstruction. It excludes:

- iSCSI CHAP password
- Camera and ONVIF passwords
- Tailscale authentication material
- Cloudflare credentials
- Cloudflare tunnel UUID where not required for reconstruction
- VM1 Tailscale `100.x` address
- Hardware serial numbers and product UUIDs


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

Replace every value shown as `<SECRET>` or `<PASSWORD>` with the value stored in the password manager.
