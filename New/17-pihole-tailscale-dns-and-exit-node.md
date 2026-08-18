# 17 — Pi-hole, Tailnet DNS, and Optional Exit Node

## Design

DNS filtering and full-traffic routing are separate controls:

| Client state | DNS path | Internet-data path |
|---|---|---|
| Tailscale connected; no exit node | Pi-hole on VM1, or public secondary | Client's current network |
| Tailscale connected; `services-vm` selected | Pi-hole on VM1, or public secondary | VM1 exit node |
| Tailscale disconnected | Local OS/network DNS | Client's current network |

Pi-hole processes DNS only. It does not carry website content, downloads, or video unless VM1 is also selected as the exit node.

The configured public secondary resolver improves availability if VM1 fails, but resolver lists are not strict primary/backup queues. A client can select `1.1.1.1` while Pi-hole is healthy, so filtering is best-effort rather than guaranteed.

## Final container topology

The initial deployment put Pi-hole on Docker network `edge` and published TCP/UDP port 53. Filtering worked, but Docker source NAT collapsed every client into one `172.x` gateway address.

The final deployment uses host networking:

```text
Tailscale client 100.x
        |
        +-- DNS :53 --> VM1 host network --> Pi-hole
        |
        +-- HTTPS --> Caddy container --> host.docker.internal:8082 --> Pi-hole UI
```

This is a deliberate exception to the shared `edge` rule. Host networking preserves client source addresses for Pi-hole reporting and group policies. Caddy remains containerized and reaches Pi-hole through Docker's host-gateway mapping.

## Preflight

Confirm host DNS port 53 is free before the first deployment:

```bash
sudo ss -lntup | grep -E '(:53[[:space:]])' || echo "Port 53 is free"
```

Do not deploy Pi-hole DHCP. The project prohibits router changes and does not need DHCP broadcast access.

## Files

Deployment directory:

```text
/home/services-vm/services/pihole/
├── compose.yaml
├── .env
└── etc-pihole/
```

`etc-pihole` contains the persistent Pi-hole database and configuration. Recreating the container does not remove it.

Protect `.env`:

```bash
chmod 600 /home/services-vm/services/pihole/.env
```

It contains:

```dotenv
PIHOLE_PASSWORD=<SECRET>
```

## Final `compose.yaml`

```yaml
name: pihole

services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    hostname: pihole
    restart: unless-stopped
    network_mode: host

    environment:
      TZ: <IANA_TIMEZONE>
      FTLCONF_webserver_api_password: ${PIHOLE_PASSWORD}
      FTLCONF_webserver_port: "8082"
      FTLCONF_dns_listeningMode: ALL
      FTLCONF_dns_upstreams: "1.1.1.1;1.0.0.1"

    volumes:
      - ./etc-pihole:/etc/pihole
```

Host networking rules:

- Do not declare `ports`; Pi-hole binds VM1 ports directly.
- Do not attach Pi-hole to `edge` or any Compose network.
- Port `53/tcp` + `53/udp` serve DNS.
- Port `8082/tcp` serves internal HTTP for Caddy.
- Caddy retains exclusive host-facing ports 80/443.

Deploy and verify:

```bash
cd /home/services-vm/services/pihole
sudo docker compose config --quiet
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=100 pihole

sudo ss -lntup | grep -E ':(53|8082)\b'
dig @<SERVICES_VM_LAN_IP> example.com
curl -I http://<SERVICES_VM_LAN_IP>:8082/admin/
```

Warnings about missing `CAP_SYS_NICE` and `CAP_SYS_TIME` are non-fatal here. Pi-hole is not providing DHCP or host time synchronization.

## Caddy integration

Caddy's Compose service requires the Linux host-gateway alias:

```yaml
services:
  caddy:
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

Inside the private wildcard site, route the Pi-hole hostname to its host-network HTTP listener:

```caddy
@pihole host pihole.example.com
handle @pihole {
    reverse_proxy host.docker.internal:8082
}
```

Do not proxy to `pihole:80`: after switching to host networking, Pi-hole is no longer discoverable through Docker DNS. Do not proxy its self-signed HTTPS listener; Caddy already provides trusted wildcard TLS.

Apply:

```bash
cd /home/services-vm/services/proxy
sudo docker compose up -d
sudo docker compose exec caddy \
  caddy validate --config /etc/caddy/Caddyfile
sudo docker compose exec caddy \
  caddy reload --config /etc/caddy/Caddyfile
```

`pihole.example.com` is private because the existing DNS-only wildcard points to VM1's Tailscale IPv4. No exact public Cloudflare Tunnel route is created.

## Tailscale DNS configuration

First prevent the DNS server from consuming its own tailnet DNS policy:

```bash
sudo tailscale set --accept-dns=false
```

In Tailscale Admin Console → **DNS**:

1. Keep MagicDNS enabled.
2. Add VM1's stable Tailscale `100.x` IPv4 as a custom global nameserver.
3. Add `1.1.1.1` as the public secondary nameserver.
4. Enable **Override DNS servers**.

Use VM1's Tailscale address, not `<SERVICES_VM_LAN_IP>`, because remote clients may not have a route to the home LAN.

Test from a Tailscale-connected client:

```bash
nslookup example.com
```

The query should appear in Pi-hole. Disconnecting Tailscale returns the client to its normal local DNS policy.

## Optional exit node

Enable persistent forwarding on VM1:

```ini
# /etc/sysctl.d/99-tailscale-exit-node.conf
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

Apply and advertise:

```bash
sudo sysctl --system
sudo tailscale set --accept-dns=false --advertise-exit-node
```

In Tailscale Admin Console → **Machines** → `services-vm` → route settings, approve **Use as exit node**. Each client then chooses `services-vm` only when full tunneling is wanted; selecting **None** returns internet traffic to the client's local route while retaining tailnet DNS.

Verify before and after selecting the exit node:

```bash
curl https://icanhazip.com
```

The public IP should change to the home connection only while `services-vm` is selected.

## Blocking policy

Exact deny entries block one hostname. A regex blocks the domain and its subdomains:

```regex
(^|\.)example\.com$
```

Pi-hole filters DNS names, not HTTPS paths or page content. DNS-over-HTTPS, Android Private DNS, hard-coded IP addresses, another VPN, and the configured public secondary resolver can bypass filtering.

After adding blocklists, update gravity:

```bash
sudo docker exec pihole pihole -g
```

Add curated lists incrementally; overlapping lists increase troubleshooting cost without proportionate coverage.

## Failure tests

Test DNS availability through the public secondary:

```bash
cd /home/services-vm/services/pihole
sudo docker compose stop pihole
```

Flush the client DNS cache and resolve a new hostname. Resolution should eventually succeed through `1.1.1.1`; delay depends on the client resolver. Restore Pi-hole:

```bash
sudo docker compose start pihole
```

Test attribution from at least two Tailscale devices. Pi-hole Query Log should show distinct `100.x` client addresses. If every client again appears as a single `172.x` address, Pi-hole has been recreated on a Docker bridge instead of host networking.

## Incident record

### Trusted domain returned HTTP 502

**Observed:** Pi-hole logs were healthy and showed web listeners, but `pihole.example.com` returned 502.

**Cause:** Caddy could not reach the selected backend. During bridge mode, both containers had to share `edge` and Caddy needed `http://pihole:80`. After the host-network migration, that name is intentionally unavailable.

**Final fix:** Pi-hole listens on host port `8082`; Caddy uses `host.docker.internal:8082`; the Caddy container defines the host-gateway alias.

### Direct VM1 address did not show the GUI

**Cause:** The initial bridge Compose published only port 53. Pi-hole's internal ports 80/443 were not host ports.

**Final state:** The UI is available internally at `http://<SERVICES_VM_LAN_IP>:8082/admin/`, but normal administration uses trusted private URL `https://pihole.example.com/admin/`.

### Every query showed the same client

**Cause:** Docker bridge NAT replaced individual Tailscale source addresses with a shared Docker gateway.

**Fix:** `network_mode: host` + no `ports` + no `edge`. Individual Tailscale clients now appear separately, enabling meaningful per-device statistics and group policies.

## Recovery checklist

```text
[ ] VM1 Tailscale is connected with accept-dns=false
[ ] Pi-hole container is running in host mode
[ ] Host TCP/UDP 53 belongs to pihole-FTL
[ ] Host TCP 8082 answers /admin/
[ ] Caddy resolves host.docker.internal via host-gateway
[ ] pihole.example.com returns trusted HTTPS
[ ] Tailnet DNS lists VM1 100.x plus 1.1.1.1
[ ] Override DNS servers is enabled
[ ] Two clients appear as distinct 100.x addresses
[ ] Exit-node selection changes public IP only when enabled
```
