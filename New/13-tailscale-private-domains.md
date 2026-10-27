# 13 — Tailscale-only Custom Domains

Private services use a normal hostname and trusted HTTPS but remain unreachable without Tailscale:

    Tailscale client → DNS-only private hostname → VM1 Tailscale IP
    → Tailscale Serve TCP 443 → 127.0.0.1:8443 → Caddy TLS → backend on edge

Tailscale Serve cannot issue a certificate for a custom domain, so it forwards raw TCP while Caddy presents the wildcard certificate. Without Tailscale, the 100.x address is not Internet-routable.

## DNS and certificate setup

Create one DNS-only wildcard record in Cloudflare:

| Type | Name | Value | Proxy |
|---|---|---|---|
| A | * | VM1 Tailscale IPv4 from tailscale ip -4 | DNS only |

Exact public tunnel records take precedence, so camera.example.com remains public while undefined names resolve privately. Remove obsolete exact private CNAMEs/tunnel routes because they shadow the wildcard.

Caddy obtains the wildcard certificate with DNS-01: it creates temporary _acme-challenge TXT records through a scoped Cloudflare token. HTTP-01/TLS-ALPN cannot work because this listener is intentionally not public. Do not substitute Funnel, a public Cloudflare route, plain HTTP, or Caddy internal CA for this design.

## Exposure policy

| Service | DNS/path | Caddy listener |
|---|---|---|
| Public web app | exact proxied tunnel route | internal HTTP :80 |
| Private web app | DNS-only wildcard → Tailscale Serve | TLS :443 via loopback :8443 |
| LAN infrastructure | no public name required | direct/restricted |

For a new private service: join its web frontend to edge, add a host matcher/upstream to the wildcard Caddy block, validate/reload Caddy, then test from a Tailscale client. No new DNS record, tunnel, or certificate is required.

Keep application authentication and Tailscale ACLs. The wildcard reveals a non-routable 100.x address; Caddy’s default wildcard handler must return 404 for unknown names.
