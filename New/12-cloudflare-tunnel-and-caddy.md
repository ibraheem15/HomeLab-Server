# 12 — Cloudflare Tunnel and Caddy for Public Frigate Access

## Objective

Publish Frigate at `https://camera.example.com` without router port forwarding or exposing VM1's public IP.

```text
Browser
  -> Cloudflare Access/edge HTTPS
  -> services-vm Cloudflare Tunnel
  -> cloudflared container
  -> Caddy container port 80
  -> Frigate authenticated port 8971
```

Cloudflare Tunnel is outbound-only. Caddy and `cloudflared` share Docker's external `edge` network; therefore the tunnel origin is `http://caddy:80`, not a LAN or Tailscale address.

## Domain delegation: Namecheap to Cloudflare

Namecheap remains the registrar. Cloudflare becomes the authoritative DNS provider.

1. Add the apex domain, such as `example.com`, to Cloudflare.
2. Review every imported record before changing nameservers. Preserve website records plus MX, SPF, DKIM, DMARC, and verification TXT records.
3. If Namecheap DNSSEC is enabled, disable it before changing nameservers.
4. In Namecheap, change **Nameservers** from Namecheap BasicDNS to the two nameservers assigned by Cloudflare.
5. Wait until Cloudflare reports the zone as **Active**. Re-enable DNSSEC through Cloudflare afterward if required.

Do not point a public `A` record at `<SERVICES_VM_LAN_IP>`, a Tailscale address, or the residential public IP for a tunnel-hosted application.

## Shared proxy directory

```text
/home/services-vm/services/proxy/
├── compose.yaml
├── Caddyfile
├── .env
├── .gitignore
└── Dockerfile       # added later for private wildcard TLS
```

The `edge` network is created once:

```bash
sudo docker network create edge
```

An “already exists” response is harmless.

## Initial proxy Compose configuration

The initial public-only deployment used the stock Caddy image:

```yaml
services:
  caddy:
    image: caddy:2-alpine
    container_name: caddy
    restart: unless-stopped
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - edge

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    command: tunnel --no-autoupdate run
    environment:
      TUNNEL_TOKEN: ${TUNNEL_TOKEN}
    depends_on:
      - caddy
    networks:
      - edge

networks:
  edge:
    external: true
    name: edge

volumes:
  caddy_data:
  caddy_config:
```

Neither container publishes a host port for the tunnel path. `cloudflared` reaches `caddy:80` through Docker networking.

`.env` contains the remotely managed tunnel token and is excluded from version control:

```dotenv
TUNNEL_TOKEN=<SECRET>
```

```gitignore
.env
```

```bash
sudo chmod 600 /home/services-vm/services/proxy/.env
```

## Frigate changes required for reverse proxying

Frigate's authenticated UI/API is port `8971`; port `5000` is unauthenticated and must not be used as a public reverse-proxy target.

In `/home/services-vm/frigate/config/config.yml`:

```yaml
tls:
  enabled: false
```

Cloudflare terminates public HTTPS. The private Caddy-to-Frigate hop uses HTTP. If Frigate TLS remains enabled, the observed error is:

```text
400 Bad Request
The plain HTTP request was sent to HTTPS port
```

Apply the configuration:

```bash
cd /home/services-vm/frigate
sudo docker compose restart frigate
curl -I http://127.0.0.1:8971
```

Any HTTP response such as `200`, `302`, or `401` proves that port `8971` now accepts HTTP. A repeated `400` means the wrong Frigate config file was edited or the container did not restart.

Frigate also joins `edge`:

```yaml
services:
  frigate:
    networks:
      - edge

networks:
  edge:
    external: true
    name: edge
```

## Hostname-based Caddy routing

Port 80 belongs to Caddy as a shared ingress listener, not to Frigate. The HTTP `Host` header selects the backend.

```caddy
http://camera.example.com {
    reverse_proxy frigate:8971
}
```

Using `http://` is intentional. A bare `camera.example.com` site address makes Caddy enable automatic HTTPS and listen on `443`; Cloudflare Tunnel is configured to send internal HTTP to port `80`.

Future public hostnames may also point to `caddy:80`, with separate site blocks. `localhost:80` is incorrect because `localhost` inside `cloudflared` means the `cloudflared` container itself.

## Tunnel creation and published route

Create one remotely managed tunnel:

```text
Zero Trust
-> Networks
-> Connectors
-> Cloudflare Tunnels
-> Create tunnel
-> cloudflared
-> Name: services-vm
```

Select Docker. Cloudflare displays a command ending in `--token eyJ...`; store only the value following `--token` in `.env`. Never paste the token into documentation or chat.

Start the connector:

```bash
cd /home/services-vm/services/proxy
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=100 cloudflared
```

After the connector is **Healthy**, add a published application route under the existing `services-vm` tunnel:

```text
Hostname:     camera.example.com
Service type: HTTP
Service URL:  caddy:80
```

The dashboard creates an exact proxied CNAME similar to:

```text
camera -> <services-vm-tunnel-UUID>.cfargotunnel.com
```

The tunnel UUID is visible as **Tunnel ID** in the tunnel details, but it is not normally copied manually. Dashboard-created routes generate their DNS records automatically.

## Duplicate tunnel incident

Two tunnels appeared: `camera` and `services-vm`. The Docker connector used the `services-vm` token; `services-vm` was active and `camera` was inactive.

Resolution:

1. Confirm `services-vm` is **Healthy** and owns `camera.example.com -> caddy:80`.
2. Confirm the inactive `camera` tunnel has no connector needed by another machine.
3. Delete the inactive tunnel.
4. Verify the `camera` CNAME still targets the `services-vm` tunnel UUID.

One tunnel can publish many applications. A service hostname is a route, not necessarily a separate tunnel.

## Security boundary

- Cloudflare Access should gate `camera.example.com` before Frigate's own login.
- Frigate port `8971` supplies the second authentication layer.
- Never proxy Frigate port `5000`.
- Do not publish Caddy ports `80`/`443` on the LAN solely for the tunnel.
- Tunnel tokens and Cloudflare API tokens are separate credentials with different purposes.

## Verification

```bash
sudo docker network inspect edge
sudo docker compose logs --tail=100 caddy
sudo docker compose logs --tail=100 cloudflared
```

The `edge` network should list `cloudflared`, `caddy`, and `frigate`. Final public URL:

```text
https://camera.example.com
```

The tunnel publishes the web application. It does not directly publish RTSP `8554` or WebRTC UDP `8555`; advanced live-view or two-way-audio behavior may use browser-compatible fallbacks.

