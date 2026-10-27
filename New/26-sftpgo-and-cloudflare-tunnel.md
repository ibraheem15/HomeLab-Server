# 26 — SFTPGo and Dedicated Tunnel

SFTPGo runs on VM2. Local state prevents SMB/network failure from corrupting SQLite or host keys; uploads use TrueNAS:

| Path | Contents |
|---|---|
| /home/baadesaba/services/sftpgo/state | SQLite, configuration, host keys |
| /mnt/vm2-business/baadesaba | user upload data |

Bind those paths explicitly in Compose and correct ownership before first start so the container can write SQLite. Verify the SMB mount before recreation; no fallback local upload directory.

Native SFTP is VM2 LAN/Tailscale port 2022. The dedicated Cloudflare Tunnel exposes only SFTPGo’s web UI; it cannot publish native SFTP. Keep tunnel token in a protected .env, inspect connector logs, and create explicit web route only.

Create the initial user with a least-privilege home under the upload root. Deferred hardening: 2FA, per-user quotas/permissions, lifecycle policy, and backup/restore test. Upstream: [SFTPGo Docker](https://docs.sftpgo.com/latest/docker/).
