# 14 — Caddy Wildcard TLS and edge Network

## Design

    Public:  cloudflared → Caddy HTTP :80 → public app
    Private: Tailscale Serve TCP :443 → 127.0.0.1:8443 → Caddy TLS → private app

Caddy needs the Cloudflare DNS module for DNS-01 wildcard certificates. Create a scoped Cloudflare API token with only Zone DNS Edit and Zone Read on this zone; put it in proxy .env with the tunnel token, mode 600, excluded from git.

## Caddy build and Compose

```dockerfile
FROM caddy:2-builder AS builder
RUN xcaddy build --with github.com/caddy-dns/cloudflare
FROM caddy:2-alpine
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

```yaml
services:
  caddy:
    build: .
    container_name: caddy
    restart: unless-stopped
    environment:
      CF_API_TOKEN: ${CF_API_TOKEN}
    ports:
      - 127.0.0.1:8443:443/tcp
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    networks: [edge]
```

Do not bind Caddy directly to the Tailscale 100.x address. It raced Tailscale at boot; Docker could reject Caddy before its restart policy applied. The loopback binding plus persistent Serve route removes that dependency:

    sudo tailscale serve --yes --bg --tcp=443 tcp://127.0.0.1:8443
    sudo tailscale serve status

Use Serve, never Funnel. Caddy still terminates TLS; forwarded peers may appear as loopback, so do not enable PROXY protocol without matching Caddy configuration.

## Caddyfile and network pattern

```caddy
http://camera.example.com {
    reverse_proxy frigate:8971
}

*.example.com {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
        resolvers 1.1.1.1
    }
    @portainer host portainer.example.com
    handle @portainer {
        reverse_proxy https://portainer:9443 {
            transport http { tls_insecure_skip_verify }
        }
    }
    handle { respond "Unknown private service" 404 }
}
```

Only web frontends join edge. Databases/caches remain on application-private networks. Apply network declarations through Compose, never manual docker network connect.

## Deploy and diagnose

    cd /home/services-vm/services/proxy
    sudo docker compose build --no-cache caddy
    sudo docker compose up -d --force-recreate caddy
    sudo docker compose exec caddy caddy list-modules | grep dns.providers.cloudflare
    sudo docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile
    sudo ss -lntp | grep :8443
    sudo tailscale serve status

For ERR_SSL_PROTOCOL_ERROR, check in order: client DNS must return VM1 100.x; Serve must own :443; Caddy must have the DNS module/token/certificate; loopback 8443 must listen. Trusted TLS plus 502 means TLS is working: inspect edge membership, backend name, port, scheme, and health.
