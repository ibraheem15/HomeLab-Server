# 18 — Netdata on VM1

Netdata is a private VM1 monitoring service. It uses host networking to observe host-level metrics, so it cannot also join edge. Keep configuration/data volumes on VM1 local storage.

```yaml
services:
  netdata:
    image: netdata/netdata:stable
    container_name: netdata
    restart: unless-stopped
    network_mode: host
    pid: host
    volumes:
      - netdataconfig:/etc/netdata
      - netdatalib:/var/lib/netdata
      - netdatacache:/var/cache/netdata
      - /:/host/root:ro,rslave
      - /etc/passwd:/host/etc/passwd:ro
      - /etc/group:/host/etc/group:ro
      - /etc/os-release:/host/etc/os-release:ro
```

Caddy cannot reach a host-network container through edge. Retain host-gateway alias in Caddy and proxy its private hostname to host.docker.internal:19999. Do not publish it publicly.

Verify container health, host listener 19999, Caddy private route, and Tailscale-only access. High Linux used memory is often filesystem cache; use available memory, swap use, OOM logs, and application latency before treating it as pressure.
