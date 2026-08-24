# 17 — AdGuard Home, Tailnet DNS, and Optional Exit Node

**Current state:** AdGuard Home is the active VM1 DNS filter. Pi-hole and the brief Technitium trial are preserved as stopped rollback options. The live cutover was verified on 2026-08-24; a full VM1 reboot test remains pending a maintenance window.

## Design

DNS filtering and full-traffic routing are separate controls:

| Client state | DNS path | Internet-data path |
|---|---|---|
| Tailscale connected; no exit node | AdGuard Home on VM1, or public secondary | Client's current network |
| Tailscale connected; `services-vm` selected | AdGuard Home on VM1, or public secondary | VM1 exit node |
| Tailscale disconnected | Local OS/network DNS | Client's current network |

AdGuard Home processes DNS only. It does not carry website content, downloads, or video unless VM1 is also selected as the exit node.

The configured public secondary resolver improves availability if VM1 fails, but resolver lists are not strict primary/backup queues. A client can select `1.1.1.1` while AdGuard Home is healthy, so filtering is best-effort rather than guaranteed.

## Current topology

```text
Tailscale/LAN client
        |
        +-- DNS TCP/UDP :53 --> VM1 host network --> AdGuard Home
        |
        +-- private HTTPS --> Caddy --> host.docker.internal:8083
                                         --> AdGuard Home UI
```

Host networking is deliberate. It preserves original LAN and Tailscale client addresses for query logs, statistics, and client policies. AdGuard Home does not join Docker network `edge`; Caddy reaches its host listener through the existing host-gateway alias.

Do not enable AdGuard Home DHCP. Static addresses remain configured inside each OS, and this project does not delegate DHCP to a DNS container.

## Persistent files

```text
/home/services-vm/services/adguardhome/
├── compose.yaml
├── compose.bridge-setup.yaml
├── conf/
│   ├── AdGuardHome.yaml
│   └── AdGuardHome.yaml.pre-host-network
└── work/
```

Both `conf` and `work` are on VM1's local NVMe-backed filesystem. The active configuration must not be edited while AdGuard Home is running because the process can overwrite manual changes.

These directories must be added to the appropriate Backrest plan. Until that backup and a representative restore are verified, the local bind mounts and the pre-host-network YAML copy are rollback aids, not a tested backup.

## Current Compose definition

```yaml
name: adguardhome

services:
  adguardhome:
    image: docker.io/adguard/adguardhome:latest
    container_name: adguardhome
    hostname: adguardhome
    restart: unless-stopped
    network_mode: host

    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf

    healthcheck:
      test:
        - CMD-SHELL
        - nslookup healthcheck.adguardhome.test. 127.0.0.1 >/dev/null 2>&1
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s
```

Host port ownership:

| Port | Protocol | Owner/use |
|---:|---|---|
| 53 | TCP + UDP | AdGuard Home DNS |
| 8083 | TCP/HTTP | AdGuard Home administration for LAN/Caddy |
| 80/443 | TCP | Caddy; never assign these to AdGuard Home |

## Initial setup record

AdGuard Home was first staged on Docker bridge networking while Technitium continued serving production port 53:

```text
192.168.1.20:3000 -> container 3000/tcp  initial wizard
192.168.1.20:8083 -> container 8083/tcp  chosen administration port
192.168.1.20:5353 -> container 53/tcp+udp staging DNS
```

The setup wizard saved the web listener as `0.0.0.0:80` despite the intended port `8083`. Docker therefore forwarded 8083 to an unused container port. The correction was made only while AdGuard Home was stopped:

```yaml
http:
  address: 0.0.0.0:8083
```

Configuration validation command:

```bash
cd /home/services-vm/services/adguardhome
sudo docker compose run --rm --no-deps \
  --entrypoint /opt/adguardhome/AdGuardHome \
  adguardhome \
  --check-config \
  -c /opt/adguardhome/conf/AdGuardHome.yaml \
  -w /opt/adguardhome/work
```

After UDP/TCP DNS and the UI worked on the staging ports, the Compose file was changed to host networking and Technitium was stopped.

## Caddy integration

Caddy's Compose service retains the Linux host-gateway alias:

```yaml
services:
  caddy:
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

Inside the existing private wildcard site:

```caddy
@adguard host adguard.example.com
handle @adguard {
    reverse_proxy host.docker.internal:8083
}
```

`adguard.example.com` is a placeholder for the real private name. It resolves through the DNS-only wildcard to VM1's Tailscale IPv4. No exact public Cloudflare Tunnel route or public DNS record is created.

Apply Caddy changes safely:

```bash
cd /home/services-vm/services/proxy
sudo docker compose exec caddy \
  caddy validate --config /etc/caddy/Caddyfile
sudo docker compose exec caddy \
  caddy reload --config /etc/caddy/Caddyfile
```

## Tailscale DNS configuration

Prevent the DNS server host from consuming its own tailnet DNS policy:

```bash
sudo tailscale set --accept-dns=false
```

Tailscale Admin Console → **DNS** retains:

1. MagicDNS enabled.
2. VM1's stable Tailscale `100.x` IPv4 as a custom global nameserver.
3. `1.1.1.1` as the public availability resolver.
4. **Override DNS servers** enabled.

Use VM1's Tailscale address, not `192.168.1.20`, because remote clients may not have a route to the home LAN. The nameserver address did not change during the Pi-hole → Technitium → AdGuard Home transitions.

## Filtering and performance

AdGuard Home loaded its default filter successfully during initial setup. Additional Pi-hole-era lists are added manually through **Filters → DNS blocklists**; do not expose private list URLs in logs or the wiki.

DNS filters block domain names, not HTTPS paths or page content. DNS-over-HTTPS configured directly in a browser, Android Private DNS, hard-coded IP addresses, another VPN, and the configured public secondary resolver can bypass filtering.

An allowed DNS response does not prove that filtering is broken. First check whether the requested name or any CNAME target exists in an enabled list, then inspect AdGuard Home's Query Log.

An initial uncached query around 45 ms was observed after cutover. Repeated cached queries were verified as working normally; measure cold and warm queries separately before changing upstreams.

## Normal verification

On VM1:

```bash
cd /home/services-vm/services/adguardhome
sudo docker compose ps
sudo docker inspect adguardhome \
  --format 'network={{.HostConfig.NetworkMode}} restart={{.HostConfig.RestartPolicy.Name}}'
sudo ss -lntup | grep -E ':(53|8083)\b'

dig +time=3 +tries=1 @127.0.0.1 example.com A
dig +time=3 +tries=1 @192.168.1.20 example.com A
dig +tcp +time=3 +tries=1 @192.168.1.20 example.com A
curl -I http://192.168.1.20:8083/
```

From a Tailscale-connected client:

```bash
nslookup example.com
curl -I https://adguard.example.com/
```

The Query Log should show the client's distinct `100.x` address, not a Docker `172.x` gateway.

## Optional exit node

The existing exit-node design is unchanged:

```ini
# /etc/sysctl.d/99-tailscale-exit-node.conf
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

```bash
sudo sysctl --system
sudo tailscale set --accept-dns=false --advertise-exit-node
```

Each client selects `services-vm` only when full tunneling is wanted. DNS can still use AdGuard Home when no exit node is selected.

## Preserved rollback services

| Service | Directory | Current state |
|---|---|---|
| Pi-hole | `/home/services-vm/services/pihole` | Stopped; restart policy `no`; `etc-pihole` and Teleporter export preserved |
| Technitium | `/home/services-vm/services/technitium` | Stopped; restart policy `no`; trial config preserved |
| AdGuard Home | `/home/services-vm/services/adguardhome` | Running; restart policy `unless-stopped`; host network |

Never start two host-network DNS containers together. Confirm the active owner and free port 53 before a rollback:

```bash
sudo ss -lntup | grep -E ':53\b' || echo 'Port 53 is free'
```

### Roll back to Pi-hole

```bash
cd /home/services-vm/services/adguardhome
sudo docker update --restart=no adguardhome
sudo docker compose stop

sudo ss -lntup | grep -E ':53\b' || echo 'Port 53 is free'

cd /home/services-vm/services/pihole
sudo docker update --restart=unless-stopped pihole
sudo docker compose start

dig @192.168.1.20 example.com A
```

Pi-hole's former UI was on port `8082`; its private Caddy route would also need to be restored from the dated pre-AdGuard Caddyfile backup.

### Roll back to Technitium

Use the same exclusivity sequence, but start `/home/services-vm/services/technitium` after proving port 53 is free. Technitium's UI uses port `5380`. Pi-hole remains the preferred historical rollback because its configuration was in service longer and has a Teleporter export.

## Recovery checklist

```text
[ ] VM1 Tailscale is connected with accept-dns=false
[ ] AdGuard Home is running with network_mode: host
[ ] Host TCP/UDP 53 belongs only to AdGuard Home
[ ] Host TCP 8083 answers the admin UI
[ ] adguard.example.com returns trusted private HTTPS
[ ] Tailnet DNS lists VM1 100.x plus 1.1.1.1
[ ] Override DNS servers and MagicDNS are enabled
[ ] Normal and blocked lookups behave as expected
[ ] Tailscale clients appear as distinct 100.x addresses
[ ] Pi-hole and Technitium remain stopped with restart policy no
[ ] VM1 reboot/persistence test is recorded after an approved maintenance window
```
