# 16 — Reusable Docker `edge` Network Pattern

## Purpose

`edge` is the shared reverse-proxy network. It lets Caddy reach web frontends by Docker service name without publishing every application port on VM1.

`edge` provides reachability, not authorization. Exposure policy remains separate:

```text
Public hostname  -> exact Cloudflare Tunnel DNS route + Cloudflare policy
Private hostname -> DNS-only wildcard to VM1 Tailscale IP + Caddy HTTPS
```

## One-time network creation

```bash
sudo docker network create edge
sudo docker network inspect edge
```

`external: true` tells each Compose project to reuse this network. Compose will not create or delete it with an application stack.

## Standard application pattern

Every multi-container application gets its own private network. Only its HTTP frontend also joins `edge`.

```yaml
name: example

services:
  app:
    image: vendor/example:VERSION
    container_name: example-app
    restart: unless-stopped
    networks:
      - example-internal
      - edge

  database:
    image: postgres:VERSION
    container_name: example-database
    restart: unless-stopped
    networks:
      - example-internal

  cache:
    image: valkey/valkey:VERSION
    container_name: example-cache
    restart: unless-stopped
    networks:
      - example-internal

networks:
  example-internal:
    name: example-internal

  edge:
    external: true
    name: edge
```

Security boundary:

| Container | Private app network | `edge` |
|---|---:|---:|
| Web/API frontend | Yes | Yes |
| Worker/ML process | Yes | No |
| Database | Yes | No |
| Cache/message queue | Yes | No |

This is deliberate dual-homing: the frontend bridges HTTP requests between Caddy and its private dependencies. Caddy cannot directly resolve or connect to the database/cache.

## Caddy routing pattern

Inside the existing `*.example.com` wildcard site:

```caddy
@example host example.example.com
handle @example {
    reverse_proxy example-app:8080
}
```

Use the container/service DNS name and the application's **container port**:

```text
Correct:   reverse_proxy example-app:8080
Incorrect: reverse_proxy localhost:8080
Incorrect: reverse_proxy <SERVICES_VM_LAN_IP>:HOST_PORT
```

Inside Caddy, `localhost` means the Caddy container. Docker DNS resolves `example-app` only when both containers share `edge`.

Backend scheme must match the application listener:

```caddy
# Plain HTTP backend
reverse_proxy example-app:8080

# HTTPS backend with a private/self-signed certificate
reverse_proxy https://example-app:8443 {
    transport http {
        tls_insecure_skip_verify
    }
}
```

Prefer an application's internal HTTP listener when traffic never leaves `edge`; Caddy already provides trusted client-facing TLS.

## Private hostname addition

For a first-level private name such as `example.example.com`:

1. Join the web container to `edge`.
2. Add its Caddy host matcher and upstream.
3. Validate and reload Caddy.
4. Test while connected to Tailscale.

No new DNS record, certificate, Cloudflare Tunnel, or API token is required. The existing DNS-only `*` record and wildcard certificate cover the hostname.

```bash
cd /home/services-vm/services/proxy
sudo docker compose exec caddy \
  caddy validate --config /etc/caddy/Caddyfile
sudo docker compose exec caddy \
  caddy reload --config /etc/caddy/Caddyfile
```

Test from a Tailscale-connected client:

```bash
dig +short example.example.com
curl -I https://example.example.com
```

Expected DNS result: VM1's `100.x` Tailscale address.

## Public hostname addition

Public exposure is explicit, never inherited from the wildcard:

1. Create an exact proxied hostname route on the existing `services-vm` Cloudflare Tunnel.
2. Point the tunnel service to Caddy's HTTP listener, normally `http://caddy:80`.
3. Add a dedicated `http://hostname` Caddy site that proxies to the frontend on `edge`.
4. Add Cloudflare Access when the application requires an outer authentication layer.

An application name is a route within the existing tunnel, not a reason to create another tunnel.

## Port publishing policy

During initial deployment, a temporary LAN port can simplify diagnosis:

```yaml
ports:
  - "8080:8080"
```

After Caddy works, remove it when direct LAN access is unnecessary. Caddy does not require host port publishing; it connects through `edge`.

Do not expose database, cache, or internal worker ports.

## Apply and verify a new service

```bash
cd /home/services-vm/services/example
sudo docker compose config --quiet
sudo docker compose up -d

sudo docker inspect example-app \
  --format '{{range $name,$config := .NetworkSettings.Networks}}{{$name}}{{"\n"}}{{end}}'

sudo docker network inspect edge
```

The frontend must list `edge` plus its internal network. Private dependencies must list only the internal network.

After changing networks, use `docker compose up -d` to recreate affected containers. Do not use `docker network connect` as the permanent configuration; manual attachment disappears after Compose recreates the container.

## Failure interpretation

| Symptom | Layer | Most likely correction |
|---|---|---|
| `ENOTFOUND database` inside app | App-private Docker DNS | Put frontend and database on the same internal network |
| Caddy cannot resolve frontend | Shared Docker DNS | Explicitly join frontend to `edge`; recreate it |
| Trusted HTTPS returns `502` | Caddy → backend | Correct container name, port, scheme, or health |
| `ERR_SSL_PROTOCOL_ERROR` | Client → Caddy TLS | Check Tailscale listener, wildcard certificate, and Caddy logs |
| Domain works without Tailscale | Exposure/DNS policy | Remove exact public/tunnel record; verify wildcard is DNS-only |

## Addition checklist

```text
[ ] Choose private vs. public exposure
[ ] Create an application-private network
[ ] Join only the web frontend to edge
[ ] Keep database/cache/worker off edge
[ ] Add Caddy hostname matcher and container port
[ ] Validate + reload Caddy
[ ] Verify DNS, TLS, backend response, and network membership
[ ] Remove unnecessary host port publishing
```
