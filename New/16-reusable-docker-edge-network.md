# 16 — Docker edge Network Pattern

edge is a shared external Docker network that lets Caddy resolve web frontends by container name. It provides reachability, not public exposure or authorization. Public/private exposure is decided by DNS, Caddy, and tunnel policy.

## Working example

Create edge once:

    sudo docker network create edge
    sudo docker network inspect edge

A normal application keeps its own internal network. Only the HTTP frontend joins both:

```yaml
name: example

services:
  app:
    image: vendor/example:VERSION
    container_name: example-app
    restart: unless-stopped
    networks: [example-internal, edge]

  database:
    image: postgres:VERSION
    container_name: example-database
    restart: unless-stopped
    networks: [example-internal]

  cache:
    image: valkey/valkey:VERSION
    container_name: example-cache
    restart: unless-stopped
    networks: [example-internal]

networks:
  example-internal:
    name: example-internal
  edge:
    external: true
    name: edge
```

Caddy can reach example-app:8080, but cannot resolve or reach the database/cache. That is the intended boundary.

```caddy
@example host example.example.com
handle @example {
    reverse_proxy example-app:8080
}
```

Use the container port and Docker DNS name. localhost means Caddy itself; a VM LAN address plus host port is unnecessary. For a self-signed HTTPS backend, use https URL plus an explicit transport exception; prefer internal HTTP when traffic stays on edge because Caddy already supplies trusted client TLS.

## Add and verify

For a private service: join only app to edge, add the matcher, validate/reload Caddy, and test from Tailscale. The wildcard DNS/certificate already covers it. For public exposure, additionally add an exact route on the existing services-vm Cloudflare Tunnel and an HTTP Caddy site.

    cd /home/services-vm/services/example
    sudo docker compose config --quiet
    sudo docker compose up -d
    sudo docker inspect example-app --format '{{range $name,$config := .NetworkSettings.Networks}}{{$name}}{{"\n"}}{{end}}'
    sudo docker network inspect edge

Remove temporary host ports after diagnosis. Never attach database, cache, or worker containers to edge. A 502 means backend network/name/port/scheme/health; an SSL protocol error is earlier, in client DNS/Tailscale/Caddy TLS.
