# 14 — Caddy Wildcard TLS, Cloudflare DNS API, and `edge` Network

## Objective

Extend the existing public-tunnel Caddy deployment so the same container also serves trusted HTTPS on VM1's Tailscale address.

```text
Public:  cloudflared -> caddy:80  -> public application
Private: Tailscale   -> caddy:443 -> private application
```

The stock `caddy:2-alpine` image does not bundle third-party DNS provider modules. The supported approach is to compile Caddy with `github.com/caddy-dns/cloudflare` using the official Caddy builder, then copy the binary into the official Alpine runtime image.

## Create a scoped Cloudflare API token

Cloudflare dashboard path:

```text
Profile icon
-> My Profile
-> API Tokens
-> Create Token
-> Create Custom Token
```

Configuration:

```text
Token name: Caddy DNS certificates

Permissions:
Zone -> DNS  -> Edit
Zone -> Zone -> Read

Zone Resources:
Include -> Specific zone -> example.com
```

Do not use the Global API Key. Avoid client-IP restrictions unless the residential public IP is static. Cloudflare displays the token secret once.

## Secret and address configuration

`/home/services-vm/services/proxy/.env`:

```dotenv
TUNNEL_TOKEN=<SECRET>
CF_API_TOKEN=<SECRET>
TAILSCALE_IP=<SERVICES_VM_TAILSCALE_IP>
```

```bash
sudo chmod 600 /home/services-vm/services/proxy/.env
```

`.env` must remain in `.gitignore`. Never include either token in diagnostics.

## Build only from original sources

`/home/services-vm/services/proxy/Dockerfile`:

```dockerfile
FROM caddy:2-builder AS builder

RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare

FROM caddy:2-alpine

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

This uses official `caddy:2-builder`, official `caddy:2-alpine`, and the Caddy DNS organization's Cloudflare source module.

An earlier example used:

```yaml
image: local/caddy-cloudflare:2
```

That string was only a local tag, not a registry image. It was removed to avoid implying that a prebuilt third-party image should be downloaded. Compose names the locally built image automatically.

## Revised Caddy Compose service

```yaml
services:
  caddy:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: caddy
    restart: unless-stopped

    environment:
      CF_API_TOKEN: ${CF_API_TOKEN}

    ports:
      - "${TAILSCALE_IP}:443:443/tcp"

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

Port `80` is not published to the host; `cloudflared` reaches it privately. Port `443` is published only on the Tailscale interface, not on VM1's LAN address.

Binding to a specific `100.x` address can fail during boot if Docker starts before Tailscale assigns that address. If Caddy is unexpectedly stopped after reboot:

```bash
systemctl is-active tailscaled
tailscale ip -4
cd /home/services-vm/services/proxy
sudo docker compose up -d caddy
```

## Combined Caddyfile

```caddy
# Public application through Cloudflare Tunnel
http://camera.example.com {
    reverse_proxy frigate:8971
}

# All private first-level subdomains share one wildcard certificate
*.example.com {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
        resolvers 1.1.1.1
    }

    @portainer host portainer.example.com
    handle @portainer {
        reverse_proxy https://portainer:9443 {
            transport http {
                tls_insecure_skip_verify
            }
        }
    }

    handle {
        respond "Unknown private service" 404
    }
}
```

`tls_insecure_skip_verify` applies only to Caddy's private connection to Portainer's self-signed backend certificate. The browser-to-Caddy connection still uses a publicly trusted wildcard certificate.

## Attach application containers to `edge`

Declaring `edge` at the top level is insufficient; the web service must explicitly join it.

Portainer example:

```yaml
services:
  portainer:
    networks:
      - default
      - edge

networks:
  default:
    name: portainer_network

  edge:
    external: true
    name: edge
```

The existing `default` network remains available for stack-specific communication. `edge` allows Caddy to resolve `portainer` through Docker DNS.

Only attach web frontends to `edge`. Keep databases and caches isolated:

```yaml
services:
  app:
    networks:
      - edge
      - internal

  database:
    networks:
      - internal

networks:
  edge:
    external: true
    name: edge
  internal:
    internal: true
```

Do not rely on `docker network connect`; manual connections disappear when Compose recreates a container.

## Build, deploy, and verify

```bash
cd /home/services-vm/services/proxy
sudo docker compose build --no-cache caddy
sudo docker compose up -d --force-recreate caddy
```

Verify the compiled module:

```bash
sudo docker compose exec caddy \
  caddy list-modules | grep dns.providers.cloudflare
```

Expected:

```text
dns.providers.cloudflare
```

Validate configuration and inspect certificate issuance:

```bash
sudo docker compose exec caddy \
  caddy validate --config /etc/caddy/Caddyfile

sudo docker compose logs --tail=150 caddy
```

Verify the host listener without exposing environment variables:

```bash
sudo docker inspect caddy \
  --format '{{json .HostConfig.PortBindings}}'

sudo ss -lntp | grep ':443'
```

Expected listener:

```text
<SERVICES_VM_TAILSCALE_IP>:443
```

## Adding future private services

The wildcard DNS record and wildcard certificate remain unchanged. Add only a matcher and upstream:

```caddy
@immich host immich.example.com
handle @immich {
    reverse_proxy immich-server:2283
}
```

Then validate and reload:

```bash
cd /home/services-vm/services/proxy
sudo docker compose exec caddy \
  caddy validate --config /etc/caddy/Caddyfile
sudo docker compose exec caddy \
  caddy reload --config /etc/caddy/Caddyfile
```

## `ERR_SSL_PROTOCOL_ERROR` incident checklist

This error occurs during TLS negotiation, before Caddy attempts to contact Portainer. A broken Docker `edge` route normally produces `502 Bad Gateway`, not an SSL protocol error.

### 1. Verify client DNS

From a Tailscale-connected client:

```bash
dig +short portainer.example.com
```

Expected: VM1's `100.x` address. Cloudflare anycast addresses, a tunnel CNAME, or a different IP indicate that an exact record is overriding the wildcard or the wildcard is incorrectly proxied.

### 2. Verify Caddy and module

```bash
cd /home/services-vm/services/proxy
sudo docker compose ps
sudo docker compose logs --tail=150 caddy
sudo docker compose exec caddy \
  caddy list-modules | grep dns.providers.cloudflare
```

Common log causes:

| Log/error | Cause | Fix |
|---|---|---|
| Module not registered | Stock image still running | Rebuild and force-recreate Caddy |
| Cloudflare authentication failure | Token missing/incorrect | Check `.env` and Compose environment mapping |
| Permission denied/zone not found | Token scope incomplete | Add `Zone.DNS:Edit`, `Zone.Zone:Read`, correct zone resource |
| Address already in use | Another process owns Tailscale `:443` | Identify listener with `ss`; remove conflicting binding |
| Cannot assign requested address | Tailscale IP absent during startup | Start/repair Tailscale, then recreate Caddy |

### 3. Inspect TLS directly

From a Tailscale-connected client:

```bash
openssl s_client \
  -connect portainer.example.com:443 \
  -servername portainer.example.com </dev/null

curl -vkI https://portainer.example.com
```

Interpretation:

```text
Timeout/refused       -> no working listener on Tailscale IP:443
No peer certificate   -> Caddy certificate/configuration failure
Trusted TLS + HTTP 502 -> TLS fixed; investigate Portainer/edge next
Trusted TLS + response -> working
```

## Current completion state

At the time this chapter was written:

- Cloudflare Tunnel + Caddy + Frigate public access was working.
- The inactive duplicate `camera` tunnel was removed.
- Wildcard DNS was added.
- Portainer was configured to join `edge`.
- The private wildcard HTTPS incident was resolved and the pattern is now used by private services.
- The `ERR_SSL_PROTOCOL_ERROR` checklist remains as the incident playbook for certificate or listener regressions.
- No Cloudflare API token value or Tailscale `100.x` address is recorded in the wiki.
