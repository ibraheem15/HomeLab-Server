# 12 — Cloudflare Tunnel and Caddy for Public Services

## Architecture and boundary

    Browser → Cloudflare HTTPS → services-vm tunnel → cloudflared → caddy:80 → Frigate:8971

Cloudflare Tunnel is outbound-only; it does not need router port forwarding. Caddy and cloudflared share external Docker network edge. Inside cloudflared, localhost is cloudflared itself, so the origin is caddy:80.

Delegate the domain’s authoritative DNS to Cloudflare while preserving imported web, MX, SPF, DKIM, DMARC, and verification records. Do not point a public A record at a LAN, Tailscale, or residential address. If moving nameservers with DNSSEC enabled, follow the registrar/Cloudflare DNSSEC sequence.

## Proxy stack

Files live at /home/services-vm/services/proxy. Create edge once:

    sudo docker network create edge

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
    networks: [edge]

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    command: tunnel --no-autoupdate run
    environment:
      TUNNEL_TOKEN: ${TUNNEL_TOKEN}
    networks: [edge]

networks:
  edge: { external: true, name: edge }
volumes: { caddy_data: {}, caddy_config: {} }
```

Store TUNNEL_TOKEN only in a mode-600 .env excluded from git. The current Caddy build is extended in [14](14-caddy-wildcard-tls-and-edge-network.md); retain its local build instead of reverting to stock Caddy.

Frigate uses authenticated HTTP on 8971 behind Caddy. Disable Frigate internal TLS, attach Frigate to edge, and route it as:

```caddy
http://camera.example.com {
    reverse_proxy frigate:8971
}
```

Port 5000 is unauthenticated and must not be the public upstream. A 400 plain-HTTP-to-HTTPS error means Frigate TLS is still enabled or the wrong config was edited.

## Tunnel route and verification

Create one remotely managed tunnel named services-vm and run its token in the Compose stack. Add public applications as routes on that tunnel:

    hostname: camera.example.com
    service:  http://caddy:80

Cloudflare creates the exact proxied CNAME. A hostname is a route, not a reason to make a new tunnel. Remove an inactive duplicate only after proving the active connector owns all routes.

    cd /home/services-vm/services/proxy
    sudo docker compose up -d
    sudo docker compose ps
    sudo docker compose logs --tail=100 caddy cloudflared
    sudo docker network inspect edge

The edge network must list Caddy, cloudflared, and the public backend. Cloudflare Access is the outer authentication layer; Frigate login remains the second layer. Native SFTP/WebRTC UDP are not published by this HTTP tunnel.
