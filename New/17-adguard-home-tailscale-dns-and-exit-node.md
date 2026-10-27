# 17 — AdGuard Home, Tailnet DNS, and Exit Node

AdGuard Home is the active VM1 DNS filter. Pi-hole and Technitium are stopped rollback options. It runs in host networking to own TCP/UDP 53 and preserve real LAN/Tailscale client addresses; it does not join edge.

DNS and exit-node routing are separate:

| Client state | DNS | General Internet traffic |
|---|---|---|
| Tailscale, no exit node | AdGuard or public secondary | client’s current network |
| Tailscale, services-vm exit selected | AdGuard or public secondary | services-vm |
| no Tailscale | local network DNS | client’s current network |

A public secondary resolver is intentionally available during VM1 outages, so filtering is best-effort rather than strict. Do not enable AdGuard DHCP; addressing remains OS-configured.

## Deployment and Caddy route

Persistent local files:

    /home/services-vm/services/adguardhome/compose.yaml
    /home/services-vm/services/adguardhome/conf/AdGuardHome.yaml
    /home/services-vm/services/adguardhome/work/

```yaml
services:
  adguardhome:
    image: docker.io/adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf
```

Port 53 TCP/UDP belongs only to AdGuard. Its HTTP UI is 8083; Caddy reaches it through host.docker.internal, requiring this Caddy setting:

```yaml
extra_hosts:
  - host.docker.internal:host-gateway
```

```caddy
@adguard host adguard.example.com
handle @adguard {
    reverse_proxy host.docker.internal:8083
}
```

If the wizard writes its listener as port 80, stop AdGuard before editing its YAML to 0.0.0.0:8083; validate configuration before restarting. Do not edit active YAML while the process runs because it can overwrite changes.

## Tailscale DNS and verification

On VM1, avoid self-DNS loops:

    sudo tailscale set --accept-dns=false

In Tailscale Admin DNS: enable MagicDNS and Override DNS servers; add VM1’s stable 100.x IP as global nameserver and 1.1.1.1 as availability resolver. Use VM1’s Tailscale IP, not its LAN address, for remote clients.

    cd /home/services-vm/services/adguardhome
    sudo docker compose ps
    sudo ss -lntup | grep -E ':(53|8083)\b'
    dig @127.0.0.1 example.com A
    dig +tcp @192.168.1.20 example.com A

From a Tailscale client, normal DNS and private HTTPS should work; query logs should show the client 100.x, not Docker 172.x. DNS-over-HTTPS, Android Private DNS, other VPNs, hard-coded addresses, and the public secondary can bypass filtering.

## Exit node and rollback

To advertise an optional exit node, persist IPv4/IPv6 forwarding, then run:

    sudo sysctl --system
    sudo tailscale set --accept-dns=false --advertise-exit-node

Approve it in Tailscale admin; each client selects it only when full tunneling is desired.

Never start two DNS containers. Before rollback, stop AdGuard and prove port 53 free. Start Pi-hole or Technitium only after that, then restore its private Caddy route if needed. A full VM1 reboot/persistence test remains a maintenance-window task.
