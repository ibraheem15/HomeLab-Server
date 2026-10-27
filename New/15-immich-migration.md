# 15 — Immich Migration and Restore

## Storage and version boundary

| Component | Location |
|---|---|
| Compose and .env | /home/services-vm/services/immich |
| PostgreSQL | /home/services-vm/services/immich/postgres |
| Existing assets | /mnt/storage-b/home/<LEGACY_USER>/immich-app/library |
| Model cache | named Docker volume |

Keep PostgreSQL on VM1 local NVMe. The existing asset library stays on Build B because VM1 does not have enough free space for a second copy. Restore with an Immich version matching the logical database dump; the recorded source dump was v3.0.3/PostgreSQL 14.19, so initial restore was pinned to v3.0.3 before later supported upgrades.

Validate the archive before touching the deployment:

    gzip -t /path/to/immich-db-backup-<TIMESTAMP>-v3.0.3-pg14.19.sql.gz
    sha256sum /path/to/immich-db-backup-<TIMESTAMP>-v3.0.3-pg14.19.sql.gz

The SQL dump is metadata and paths; it is not a media backup.

## Environment and networks

```dotenv
UPLOAD_LOCATION=/mnt/storage-b/home/<LEGACY_USER>/immich-app/library
DB_DATA_LOCATION=/home/services-vm/services/immich/postgres
IMMICH_VERSION=v3.0.3
DB_PASSWORD=<SECRET>
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```

DB_DATA_LOCATION must end in /postgres, never the deployment directory. PostgreSQL will legitimately chown the directory it receives to its internal UID and can make the whole deployment inaccessible if the path is wrong.

Only the frontend is dual-homed:

```yaml
services:
  immich-server:
    networks: [immich-internal, edge]
  database:
    networks: [immich-internal]
  redis:
    networks: [immich-internal]
  immich-machine-learning:
    networks: [immich-internal]
networks:
  immich-internal: { name: immich-internal }
  edge: { external: true, name: edge }
```

Caddy routes to immich-server:2283. PostgreSQL, Valkey, and ML must never join edge.

## Recovery lessons

If initdb says its data directory is not empty, stop Compose, quarantine the failed postgres directory with a dated rename, create a new empty postgres directory with the normal VM1 owner, then start database alone and wait for ready-to-accept-connections. Do not delete the failed directory first.

If the deployment directory became UID 999, DB_DATA_LOCATION was likely the deployment root. Stop using absolute Compose paths, quarantine the broken directory, recreate deployment plus postgres child, copy only compose.yaml and .env back, correct the path, and inspect the resolved mount:

    docker compose config | grep -B2 -A3 /var/lib/postgresql/data

If Immich reports ENOTFOUND database while PostgreSQL is healthy, all services do not share immich-internal. Recreate from Compose rather than manually attaching networks. Persist vm.overcommit_memory=1 for Valkey.

## Finish and verify

Add the private Caddy matcher to the existing wildcard block:

```caddy
@immich host immich.example.com
handle @immich { reverse_proxy immich-server:2283 }
```

After proxy verification, remove unnecessary direct host port 2283. Confirm prior users/albums, old photos/videos, new upload, database files only in local postgres, and tailnet-only access. Keep quarantined directories until a new verified backup and asset checks pass. Use [Immich backup/restore](https://immich.app/docs/administration/backup-and-restore/) for version-specific restore steps.
