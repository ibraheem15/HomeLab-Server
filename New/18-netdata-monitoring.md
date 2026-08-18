# 18 — Netdata Monitoring on VM1

## Purpose

Netdata provides host, Docker, CPU, memory, disk, filesystem, and network visibility for `services-vm`. It monitors VM1 only; Proxmox and Build B require separate agents if host-level monitoring is added later.

## Networking decision

Docker Compose rejects a service that declares both:

```yaml
network_mode: host
networks:
  - edge
```

These modes are mutually exclusive. Netdata uses `network_mode: host` for complete host visibility. It therefore does **not** join `edge`; Caddy reaches it through `host.docker.internal:19999`, using the same host-gateway pattern as Pi-hole.

## Deployment layout

```text
/home/services-vm/services/netdata/
└── compose.yaml
```

Final Compose structure:

```yaml
name: netdata

services:
  netdata:
    image: netdata/netdata:latest
    container_name: netdata
    hostname: services-vm
    restart: unless-stopped
    pid: host
    network_mode: host

    cap_add:
      - SYS_PTRACE
      - SYS_ADMIN
    security_opt:
      - apparmor:unconfined

    volumes:
      - netdataconfig:/etc/netdata
      - netdatalib:/var/lib/netdata
      - netdatacache:/var/cache/netdata
      - /etc/passwd:/host/etc/passwd:ro
      - /etc/group:/host/etc/group:ro
      - /etc/os-release:/host/etc/os-release:ro
      - /var/log:/host/var/log:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro

volumes:
  netdataconfig:
  netdatalib:
  netdatacache:
```

Deploy:

```bash
cd /home/services-vm/services/netdata
sudo docker compose config --quiet
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=100 netdata
```

Direct listener:

```text
http://192.168.1.20:19999
http://<VM1-TAILSCALE-IP>:19999
```

## Private Caddy route

Caddy must already contain:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

Caddyfile route:

```caddy
@netdata host netdata.domain.com
handle @netdata {
    reverse_proxy host.docker.internal:19999
}
```

`netdata.domain.com` remains private because the DNS-only wildcard resolves it to VM1's Tailscale IP. Do not create a public Tunnel route for it.

## Persistence

The three named volumes survive container deletion and recreation:

```bash
sudo docker volume ls | grep netdata
```

`docker compose down` preserves them. `docker compose down -v` deletes them and must not be used unless resetting Netdata intentionally. Container deletion ≠ data deletion; volume deletion removes configuration/history/cache.

## Memory interpretation

Linux uses otherwise-idle RAM for filesystem cache. Proxmox may display VM memory near 100% even when applications are not under pressure.

The authoritative value is `available`, not `free`:

```bash
free -h
```

Observed VM1 state:

```text
23 GiB total; 3.8 GiB used; 19 GiB buff/cache; 19 GiB available; swap unused
```

This represented healthy reclaimable cache, not memory exhaustion. Investigate only when available memory falls persistently, swap use rises, or the kernel logs OOM events.
