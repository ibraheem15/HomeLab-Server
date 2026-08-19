# 26 — SFTPGo and Dedicated Cloudflare Tunnel

## Persistence model

```text
VM2 NVMe
└── /home/baadesaba/services/sftpgo/state
    SQLite database, configuration state, SSH host keys

TrueNAS SATA SSD
└── /mnt/vm2-business
    SFTPGo user files and SFTPGo data tree
```

This keeps application state/database local while bulk uploads use the expandable TrueNAS dataset.

## Compose deployment

`/home/baadesaba/services/sftpgo/compose.yaml`:

```yaml
services:
  sftpgo:
    image: drakkan/sftpgo:v2
    container_name: sftpgo
    restart: unless-stopped
    stop_grace_period: 40s
    environment:
      SFTPGO_GRACE_TIME: "30"
      TZ: Asia/Karachi
    ports:
      - "8080:8080"
      - "2022:2022"
    volumes:
      - ./state:/var/lib/sftpgo
      - type: bind
        source: /mnt/vm2-business
        target: /srv/sftpgo
        bind:
          create_host_path: false

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: drive_cloudflared
    restart: unless-stopped
    command:
      - tunnel
      - --no-autoupdate
      - run
      - --token
      - ${CF_TUNNEL_TOKEN}
    depends_on:
      - sftpgo
```

The token is stored only in mode-600 `.env`:

```dotenv
CF_TUNNEL_TOKEN=<SECRET>
```

## SQLite permission incident

Initial logs showed:

```text
error initializing data provider: unable to open database file
```

The SFTPGo image runs as UID/GID 1000. Its bind-mounted local state directory needed matching ownership:

```bash
sudo docker compose down
sudo chown -R 1000:1000 /home/baadesaba/services/sftpgo/state
sudo chmod 750 /home/baadesaba/services/sftpgo/state
sudo docker compose up -d
```

After correction, SQLite initialized and SFTPGo listened on HTTP 8080 and SFTP 2022.

## Initial SFTPGo setup

The first WebAdmin login created a dedicated administrator. A separate regular account `uploads` uses:

```text
Filesystem: Local
Home: /srv/sftpgo/data/uploads
Permissions: list, download, upload, overwrite, delete, rename, create directories
SFTPGo quota: unlimited/0
```

TrueNAS's 100 GiB dataset quota is the authoritative capacity limit. Upload/download/delete testing through `/web/client` confirmed that files appear below `/mnt/vm2-business/data/uploads`, not on VM2's root disk.

## Cloudflare tunnel

A dedicated remotely managed tunnel named `drive-vm` runs on VM2. It is intentionally independent of VM1/Caddy and VM1's `services-vm` tunnel.

Published application route:

```text
Public hostname: drive.<domain>
Service:         http://sftpgo:8080
```

Use `sftpgo:8080`, not `localhost:8080`; inside the `cloudflared` container, localhost refers to `cloudflared` itself. Both containers share Compose network `sftpgo_default`.

Cloudflare automatically supplies public HTTPS. Origin traffic remains HTTP inside VM2's Docker bridge, which is correct.

## Tunnel incident

The public hostname initially showed `Unknown private service`. Checks confirmed both containers shared `sftpgo_default`. The published application route was correct; Cloudflare propagation required additional time. The hostname then became operational without further changes.

High-signal checks:

```bash
cd /home/baadesaba/services/sftpgo
sudo docker compose ps
sudo docker compose logs --tail=100 cloudflared
sudo docker inspect sftpgo --format '{{json .NetworkSettings.Networks}}'
sudo docker inspect drive_cloudflared --format '{{json .NetworkSettings.Networks}}'
```

Origin test from the Compose network:

```bash
sudo docker run --rm --network sftpgo_default \
  curlimages/curl:latest -sS -I http://sftpgo:8080/web/client
```

## Deferred security work

Cloudflare Access was explicitly skipped. Therefore `/web/client` and `/web/admin` are public login endpoints protected only by SFTPGo authentication.

Before distributing accounts broadly:

1. Enable TOTP 2FA for the SFTPGo administrator and file users.
2. Review SFTPGo defender/brute-force and rate-limit settings.
3. Store recovery codes offline.
4. Back up local SFTPGo state plus TrueNAS user data.
5. Never reuse TrueNAS SMB, SFTPGo admin, or SFTPGo file-user passwords.
