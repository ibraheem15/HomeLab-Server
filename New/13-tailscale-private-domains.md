# 13 — Tailscale-Only Custom Domains and Wildcard DNS

## Requirement

Administrative services such as Portainer must use a normal custom hostname and trusted HTTPS, but remain unreachable unless the client is connected to the tailnet.

```text
Required:  https://portainer.example.com
Allowed:   Tailscale clients only
Rejected:  Public Cloudflare Tunnel route
Rejected:  Router port forwarding
Rejected:  Browser certificate warnings
```

Cloudflare Tunnel is unsuitable for this specific boundary because it makes the hostname reachable from Cloudflare's public edge. Cloudflare Access adds authentication but does not make Tailscale connectivity mandatory.

Tailscale Serve's HTTPS termination is limited to the tailnet's `*.ts.net` names, so it cannot itself issue a certificate for `portainer.example.com`. The selected design instead uses Serve as a raw TCP forwarder; Caddy receives the original TLS stream and supplies the custom-domain wildcard certificate.

## Final architecture

```text
Tailscale-connected client
  -> public DNS lookup
  -> portainer.example.com = VM1's stable 100.x address
  -> encrypted Tailscale path
  -> Tailscale Serve raw TCP :443
  -> 127.0.0.1:8443
  -> Caddy container port 443
  -> Portainer container on edge network
```

Without Tailscale, the browser can resolve the hostname but cannot route to the `100.64.0.0/10` tailnet address.

## Wildcard DNS

Rather than adding every private service to Cloudflare, create one DNS-only wildcard:

```text
Type:         A
Name:         *
IPv4 address: <services-vm Tailscale IPv4>
Proxy status: DNS only — gray cloud
TTL:          Auto
```

Obtain VM1's address with:

```bash
tailscale ip -4
```

The wildcard covers undefined names:

```text
portainer.example.com
immich.example.com
vault.example.com
jellyfin.example.com
```

Cloudflare-specific records take precedence over wildcard records. Therefore the existing exact public record remains valid:

```text
CNAME camera -> <tunnel-UUID>.cfargotunnel.com  Proxied
```

while undefined administrative names resolve privately:

```text
A * -> <Tailscale-IP>  DNS only
```

Before relying on the wildcard, remove any obsolete exact `portainer` CNAME or tunnel route; an exact record shadows the wildcard.

## Why a DNS challenge is required

A public certificate authority normally validates a site by connecting to it over HTTP/TLS. A Tailscale-only listener is deliberately unreachable from the public Internet, so HTTP-01 and TLS-ALPN-01 validation cannot work.

DNS-01 instead proves domain control by creating a temporary TXT record:

```text
_acme-challenge.example.com
```

Caddy creates and removes that TXT record through a scoped Cloudflare API token. No inbound port, router change, or public tunnel is required. The resulting wildcard certificate is trusted by ordinary browsers.

Alternatives rejected:

| Option | Why not selected |
|---|---|
| Tailscale Serve HTTPS termination | Trusted HTTPS, but only `*.ts.net`; selected raw TCP mode leaves custom-domain TLS termination to Caddy |
| Cloudflare Tunnel + Access | Public edge remains reachable without Tailscale |
| Caddy `tls internal` | Requires installing Caddy's private CA on every client |
| Plain HTTP over Tailscale | WireGuard encrypts transport, but the browser lacks HTTPS identity and secure-origin behavior |
| Cloudflare Origin certificate | Trusted by Cloudflare, not by a browser connecting directly over Tailscale |
| Nginx Proxy Manager | GUI simplifies DNS challenge, but duplicates the existing Caddy reverse proxy |

## Public vs. private service policy

| Service class | DNS | Network path | Reverse proxy |
|---|---|---|---|
| Public application | Exact proxied CNAME created by tunnel route | Cloudflare Tunnel | Caddy port 80 |
| Private admin application | Wildcard DNS-only A to `100.x` | Tailscale Serve raw TCP `:443` -> loopback `:8443` | Caddy port 443 |
| LAN-only infrastructure | No public DNS required | LAN | Direct or separately restricted |

Examples:

```text
camera.example.com    -> public tunnel -> Caddy -> Frigate
portainer.example.com -> Tailscale IP  -> Caddy -> Portainer
```

## Future service workflow

For a new private first-level subdomain:

1. Attach only the application's web container to Docker network `edge`.
2. Add a hostname matcher and upstream in the wildcard Caddy site.
3. Reload Caddy.

No new Cloudflare DNS record, tunnel route, or certificate is required.

For a new public service:

1. Add an exact hostname block on Caddy's internal HTTP listener.
2. Add a published application route to the existing `services-vm` tunnel.
3. Add Cloudflare Access if the application is not intentionally anonymous.

## Security implications

- A DNS-only wildcard reveals VM1's `100.x` address publicly, but that address is not Internet-routable.
- The wildcard causes misspelled/undefined subdomains to resolve to VM1. Caddy's default handler must return `404` rather than exposing a default backend.
- Tailscale ACLs remain the authorization layer controlling which tailnet users/devices may reach VM1.
- Caddy's host publication is loopback-only (`127.0.0.1:8443`); do not change it to `0.0.0.0` as a shortcut.
- Use `tailscale serve`, never `tailscale funnel`, for this private path.
- A wildcard certificate protects transport and hostname identity; it does not authorize users.
- Keep application authentication enabled, especially for Portainer and Vaultwarden.
