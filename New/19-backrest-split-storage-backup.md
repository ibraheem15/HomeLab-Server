# 19 — Backrest Backup with Split VM1 and iSCSI Storage

## Backup model

Application state is split across two storage tiers:

| Data | Source | Backup rule |
|---|---|---|
| Immich PostgreSQL | VM1 local SSD | Do not copy the live PostgreSQL directory |
| Immich Compose/config | VM1 local SSD | Back up |
| Immich assets + generated DB dumps | Build B iSCSI | Back up |
| Vaultwarden active deployment/data | VM1 local SSD | Back up |
| Old Vaultwarden directory on Build B | Build B iSCSI | Exclude; obsolete duplicate |
| Personal files | Build B iSCSI | Back up |
| Other VM1 service folders | VM1 local SSD | Not in the current plan |

The external 500 GB HDD contains the existing Restic repository. Reusing the repository preserves prior snapshots and enables content-defined deduplication. A new snapshot records a new filesystem state; unchanged chunks are referenced rather than stored again.

## Deployment paths

```text
Backrest Compose: /home/services-vm/services/backrest/compose.yaml
Repository mount: /mnt/backup-seagate
iSCSI mount:      /mnt/storage-b
```

Recommended bind mounts:

```yaml
services:
  backrest:
    image: ghcr.io/garethgeorge/backrest:latest
    container_name: backrest
    restart: unless-stopped

    environment:
      BACKREST_DATA: /data
      BACKREST_CONFIG: /config/config.json
      XDG_CACHE_HOME: /cache
      TMPDIR: /tmp
      TZ: Asia/Karachi

    volumes:
      - ./data:/data
      - ./config:/config
      - ./cache:/cache
      - ./tmp:/tmp

      - /mnt/backup-seagate:/repo
      - /mnt/storage-b/home/ibraheem/Personal:/source/personal:ro
      - /mnt/storage-b/home/ibraheem/immich-app/library:/source/immich-data:ro
      - /home/services-vm/services/immich:/source/immich-local:ro
      - /home/services-vm/services/vaultwarden:/source/vaultwarden:ro

    ports:
      - "9898:9898"

    labels:
      - diun.enable=true
```

If Backrest is routed through Caddy, attach its web frontend to `edge`; this does not alter repository/source mounts.

## Repository safety gate

The external disk root contains both the Restic `config` file and a sentinel:

```text
/mnt/backup-seagate/config
/mnt/backup-seagate/is_mounted
```

Configure the repository snapshot-start hook to fail when either is absent:

```bash
test -f /repo/is_mounted && test -f /repo/config
```

Use condition `CONDITION_SNAPSHOT_START` and fatal error handling. This prevents Backrest from initializing or writing into the empty VM1 mount directory when the USB filesystem is absent.

## Existing repository

Repository URI inside Backrest:

```text
/repo
```

Use the existing repository password. If Backrest proposes initializing a new repository, stop immediately. That means `/repo` is empty, mounted incorrectly, or points at the wrong container path.

Successful repository discovery is confirmed by listing snapshots and running the repository check. Do not initialize over the existing root.

## Immich database rule

Exclude the live PostgreSQL tree:

```text
/source/immich-local/postgres
/source/immich-local/postgres/**
```

Reason: copying a running PostgreSQL data directory is not a transaction-consistent backup. Immich-generated compressed SQL backups already reside under the backed-up asset tree:

```text
/source/immich-data/backups/*.sql.gz
```

The working restore used:

```text
immich-db-backup-<TIMESTAMP>-v3.0.3-pg14.19.sql.gz
```

Therefore the recoverable pair is SQL dump + complete Immich asset tree, not a raw live database directory.

## Capacity and deduplication

External repository state before the new plan:

```text
458 GiB filesystem
154 GiB used
281 GiB available
```

Restic compares content chunks against the existing repository. Moving a source path or changing the Backrest plan can create a logically new snapshot without physically duplicating every unchanged file. New/changed chunks consume space; snapshot metadata is comparatively small.

## Verification

After each manual run:

```bash
sudo docker compose -f /home/services-vm/services/backrest/compose.yaml ps
df -h /mnt/backup-seagate
```

In Backrest, confirm plan success, review the snapshot contents, then run repository check periodically. A backup is not proven until a test restore succeeds.
