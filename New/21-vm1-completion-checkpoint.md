# 21 — VM1 Checkpoint

VM1 is the active service platform: Docker, Caddy/cloudflared, Frigate, Immich, AdGuard Home, and the sole Build B iSCSI initiator. Local NVMe owns configuration and databases; Build B owns media; external HDD owns offline backups.

Direct host ports are optional diagnostics. Normal web access is Caddy via Cloudflare Tunnel for public routes or Tailscale Serve for private routes. Verify storage mount before media-dependent Compose recreation, keep Caddy loopback-only, and keep databases off network storage.

Remaining phases: VM3 configuration, SFTPGo hardening, GPU passthrough only if measured need justifies it, and an intentional iSCSI-to-NFS migration if desired. This checkpoint is superseded by 00-current-state for live facts.
