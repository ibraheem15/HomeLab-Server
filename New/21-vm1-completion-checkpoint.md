# 21 — VM1 Completion Checkpoint

## Completed platform

`services-vm` is the sole application VM and iSCSI initiator. Its current operating model is:

```text
VM1 local SSD → Debian, Docker, Compose, application configuration, databases
Build B iSCSI → bulk media, Immich assets, Frigate recordings, Personal data
USB HDD      → normally-off Restic repository for manual Backrest sessions
edge network → Caddy-to-frontend routing only
Tailscale    → administration, private HTTPS names, Pi-hole DNS, optional exit node
Cloudflare   → explicit public route for camera.domain.com only
```

## Services covered in this build stage

| Service | State | Persistent data rule | Access model |
|---|---|---|---|
| Frigate | Operational | Config/DB local; recordings on iSCSI | Public `camera.domain.com` through Tunnel + Caddy |
| Immich | Operational/restored | PostgreSQL local; assets on iSCSI | Private wildcard HTTPS through Caddy |
| Caddy + cloudflared | Operational | Proxy config local | Shared ingress; one active tunnel |
| Portainer | Integrated | Local Docker volume/data | Private wildcard HTTPS |
| Pi-hole | Operational | `./etc-pihole` local | Tailnet DNS; private UI; public resolver available |
| Tailscale exit node | Advertised | Host sysctl/config | Client opt-in only |
| Netdata | Documented deployment | Named Docker volumes | Host port 19999/private Caddy route |
| Backrest | Existing repo verified | App state local; repository on removable HDD | Manual sessions only |

Vaultwarden and Jellyfin remain outside this checkpoint unless separately migrated and validated later.

## Direct ports vs. domain access

Caddy accesses containers through Docker network `edge`; this does not automatically expose their ports on VM1.

```text
service.domain.com → Caddy → edge → container:internal-port
TAILSCALE_IP:port  → VM1 host port → published Docker port
```

Therefore a domain may work while `TAILSCALE_IP:port` fails. Direct access requires a Compose `ports` mapping:

```yaml
ports:
  - "2283:2283"
```

This binds all VM1 interfaces. To expose only through Tailscale, bind the stable Tailscale IP explicitly:

```yaml
ports:
  - "100.x.y.z:2283:2283"
```

Use `http://` unless the application itself provides TLS. Caddy's HTTPS certificate does not make the application's raw host port HTTPS.

## Mandatory invariants

1. `/mnt/storage-b` must be mounted before starting any service with iSCSI-backed bind mounts.
2. PostgreSQL/SQLite databases stay on VM1 local SSD; bulk media stays on iSCSI.
3. Only frontends join `edge`; databases and caches remain application-private.
4. Public access requires an explicit Tunnel DNS route; the wildcard remains DNS-only/private.
5. The backup HDD remains unmounted and powered off outside manual backup windows.

## Baseline health check

```bash
findmnt /mnt/storage-b
sudo iscsiadm -m session -P 1
sudo docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
sudo docker network inspect edge
tailscale status
free -h
df -hT / /mnt/storage-b
systemctl --failed
```

Expected: iSCSI logged in; storage mounted `rw`; required containers healthy; no failed units; swap idle under normal load; substantial `MemAvailable` even if Proxmox reports high used RAM.

## Restart dependency order

```text
storage-b target ready
→ VM1 network + Tailscale
→ open-iscsi login
→ /mnt/storage-b mounted
→ application stacks
→ Caddy/cloudflared
```

Never let Docker silently create empty local directories where iSCSI bind sources should be. Use bind long syntax with `create_host_path: false` for critical media paths where supported.

## Remaining project phases

- VM2: Debian + SFTPGo + Cloudflare Tunnel/Access.
- VM3: co-owner environment and separate Tailscale organization; admin boundary unresolved.
- Build A 1 TB SATA SSD: intentionally untouched until a storage-pool decision is made.
- VM1 later: migrate/validate Vaultwarden and Jellyfin, then update this checkpoint.
- Maintenance: firmware/kernel updates, Intel NIC stability observation, restore testing, and periodic repository checks.
