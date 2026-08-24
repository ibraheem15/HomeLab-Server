# 14 — Caddy Wildcard TLS, Cloudflare DNS API, and `edge` Network

## Objective

Extend the existing public-tunnel Caddy deployment so the same container also serves trusted HTTPS on VM1's Tailscale address.

```text
Public:  cloudflared -> caddy:80  -> public application
Private: Tailscale Serve TCP :443 -> 127.0.0.1:8443 -> caddy:443 -> private application
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
      - "127.0.0.1:8443:443/tcp"

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

Port `80` is not published to the host; `cloudflared` reaches it privately. Caddy's TLS port is published only on host loopback `127.0.0.1:8443`, so neither LAN nor internet clients can reach that listener directly. Tailscale Serve owns tailnet TCP port `443` and forwards raw TCP to Caddy. Caddy still terminates TLS and presents the wildcard certificate.

Configure the persistent raw TCP forwarder on `services-vm`:

```bash
sudo tailscale serve --yes --bg \
  --tcp=443 \
  tcp://127.0.0.1:8443
sudo tailscale serve status
```

Use `serve`, never `funnel`: Serve is tailnet-only. The background configuration is retained by Tailscale and resumes when Tailscale returns after a reboot. This design does not need a separate proxy startup script or per-stack systemd unit.

### 2026-08 power-outage reliability correction

During an uncontrolled power loss, Caddy received external `SIGTERM`, performed a graceful shutdown, and exited with status `0`; backend connection-refused and Docker-DNS errors in the same log were consequences of Frigate being unavailable during shutdown/startup, not a Caddy crash. The host restarted before public internet connectivity returned.

The previous Compose mapping bound Docker directly to VM1's Tailscale `100.x` address. That introduced a boot race: Docker could try to create Caddy before the Tailscale address existed, and Docker restart policies cannot recover a container that was never created successfully. The live mapping was therefore changed on 2026-08-25 to loopback `127.0.0.1:8443`, with Tailscale Serve providing persistent tailnet TCP `443` forwarding. Private HTTPS was verified working after the cutover. Full VM reboot persistence remains unverified until an approved maintenance window.

Raw TCP forwarding has two operational consequences:

- Caddy continues to terminate TLS, so the existing custom-domain wildcard certificate and SNI routing remain unchanged.
- Caddy may see the forwarded peer as loopback rather than the original Tailscale client. Do not enable PROXY protocol unless Caddy is explicitly configured to parse it.

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

sudo ss -lntp | grep ':8443'
sudo tailscale serve status
```

Expected local mapping and Serve route:

```text
127.0.0.1:8443 -> caddy:443/tcp
tailnet TCP :443 -> tcp://127.0.0.1:8443
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
| Loopback `8443` already in use | Another host process owns Caddy's local forwarding port | Identify the owner with `ss`; do not expose Caddy on `0.0.0.0` as a shortcut |
| No Tailscale Serve route on `:443` | Serve configuration is absent or Tailscale is not running | Check `tailscale serve status` and `systemctl status tailscaled`; restore the documented raw TCP route |
| `cannot assign requested address` in historical Caddy logs | Old Compose mapping bound directly to a Tailscale `100.x` address before it existed | Keep the current loopback mapping plus Tailscale Serve; do not restore the direct `100.x:443` bind |

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
- Caddy TLS is bound to loopback `127.0.0.1:8443`; persistent Tailscale Serve raw TCP forwarding supplies tailnet port `443`.
- Private HTTPS was verified after the 2026-08-25 cutover; full reboot recovery remains pending a maintenance-window test.
- The `ERR_SSL_PROTOCOL_ERROR` checklist remains as the incident playbook for certificate or listener regressions.
- No Cloudflare API token value or Tailscale `100.x` address is recorded in the wiki.
