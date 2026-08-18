# 15 — Immich Migration and Database Recovery

## Final architecture

Immich was migrated rather than installed as an empty service. The existing asset tree remains on Build B through iSCSI; PostgreSQL and Compose state live on VM1's local SSD.

| Component | Final location | Reason |
|---|---|---|
| Compose and `.env` | `/home/services-vm/services/immich` | Local, independent of storage network |
| PostgreSQL | `/home/services-vm/services/immich/postgres` | Databases must not use network block storage |
| Asset root | `/mnt/storage-b/home/<LEGACY_USER>/immich-app/library` | Preserve the existing 89 GiB library without copying |
| Database backup | Asset root `backups/` directory | Used by Immich onboarding restore |
| Model cache | Docker named volume `model-cache` | Regenerable data |

The migration source contained approximately 89 GiB of assets and 585 MiB of raw PostgreSQL data. VM1 had approximately 74 GiB free, enough for PostgreSQL but not for a second asset copy.

## Backup selected

The newest verified logical backup was:

```text
immich-db-backup-<TIMESTAMP>-v3.0.3-pg14.19.sql.gz
```

The filename identifies the source as Immich `v3.0.3` with PostgreSQL `14.19`. The first deployment was therefore pinned to `v3.0.3`; using `release` during restoration could introduce an avoidable version mismatch.

Before modifying the deployment:

```bash
gzip -t /path/to/immich-db-backup-<TIMESTAMP>-v3.0.3-pg14.19.sql.gz
sha256sum /path/to/immich-db-backup-<TIMESTAMP>-v3.0.3-pg14.19.sql.gz
```

The database dump contains metadata and file paths, not the photos/videos. A complete recovery requires both the SQL dump and the matching asset directories.

## Environment layout

Relevant `.env` entries:

```dotenv
UPLOAD_LOCATION=/mnt/storage-b/home/<LEGACY_USER>/immich-app/library
DB_DATA_LOCATION=/home/services-vm/services/immich/postgres
TZ=<IANA_TIMEZONE>
IMMICH_VERSION=v3.0.3
DB_PASSWORD=<SECRET>
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```

The old database password was retired. `DB_STORAGE_TYPE: HDD` was removed because the new PostgreSQL directory is on VM1's SSD.

Critical distinction:

```text
.../services/immich          = deployment directory
.../services/immich/postgres = PostgreSQL bind mount
```

Missing `/postgres` from `DB_DATA_LOCATION` causes PostgreSQL to treat the entire deployment directory as its data directory.

## Compose network design

Only the HTTP frontend is dual-homed:

```yaml
services:
  immich-server:
    networks:
      - immich-internal
      - edge

  immich-machine-learning:
    networks:
      - immich-internal

  redis:
    networks:
      - immich-internal

  database:
    networks:
      - immich-internal

networks:
  immich-internal:
    name: immich-internal

  edge:
    external: true
    name: edge
```

Result:

```text
Caddy --edge-- immich-server --immich-internal-- PostgreSQL / Valkey / ML
```

PostgreSQL, Valkey, and machine learning never join `edge`. Caddy reaches `immich-server:2283` through Docker DNS.

## Incident 1 — PostgreSQL directory not empty

Initial PostgreSQL logs reported:

```text
initdb: error: directory "/var/lib/postgresql/data" exists but is not empty
```

Immich then repeatedly reported:

```text
Error: getaddrinfo ENOTFOUND database
```

The server error was downstream: PostgreSQL was restarting, so its Docker DNS record was unavailable.

Recovery preserved the failed directory instead of deleting it:

```bash
sudo docker compose down
sudo mv postgres postgres.failed-20260818
sudo install -d -o services-vm -g services-vm -m 750 postgres
```

The replacement directory was verified empty before starting only the database:

```bash
sudo docker compose up -d database
sudo docker compose logs --tail=100 database
```

Expected terminal state:

```text
database system is ready to accept connections
```

## Incident 2 — Deployment directory became UID 999

`DB_DATA_LOCATION` was accidentally set to the deployment directory rather than its `postgres` child. The PostgreSQL container correctly changed its mounted directory to internal UID `999`, which made this fail for the normal user:

```text
cd: Permission denied: 'immich'
```

This was not a general home-directory permission failure. The exact deployment directory had become the PostgreSQL data directory.

Recovery sequence:

1. Stop Compose using absolute `--project-directory` and `-f` paths.
2. Rename the affected directory to `immich.bad-db-path-20260818`.
3. Recreate `immich/` and `immich/postgres/` with `services-vm` ownership.
4. Copy only `compose.yaml` and `.env` from the quarantined directory.
5. Correct `DB_DATA_LOCATION`, validate the resolved mount, then restart.

Never recursively change ownership of a valid PostgreSQL data directory. PostgreSQL requires ownership by its container UID.

Resolved-mount verification:

```bash
docker compose config |
grep -B2 -A3 '/var/lib/postgresql/data'
```

Required source:

```text
/home/services-vm/services/immich/postgres
```

## Incident 3 — Healthy PostgreSQL but `ENOTFOUND database`

After PostgreSQL became healthy, Immich still failed to resolve `database`. Cause: the frontend and database did not share a Docker network.

Fix: explicitly attach all four services to `immich-internal`, then attach only `immich-server` to `edge`. Recreating containers applied the network aliases:

```bash
sudo docker compose down
sudo docker compose up -d
sudo docker exec immich_server getent hosts database
sudo docker exec immich_server getent hosts redis
```

Both names must return container IP addresses. Manual `docker network connect` is not persistent and was not used.

## Valkey kernel warning

Valkey warned that memory overcommit was disabled. Persist the recommended setting:

```bash
echo 'vm.overcommit_memory=1' |
sudo tee /etc/sysctl.d/99-valkey.conf
sudo sysctl --system
```

This warning did not cause the Docker DNS failure; it prevents background-save failures under memory pressure.

## Onboarding restore and profile warning

The onboarding storage check confirmed `encoded-video`, `library`, `upload`, `profile`, `thumbs`, and `backups` were readable/writable. It reported missing files only under `profile`.

`profile` contains account avatars; original assets reside in `library` and `upload`. Adding a `.immich` marker cannot repair a missing database-referenced avatar. The marker validates a mount; it is not a substitute for a named file.

Because both original-asset directories passed, restoration continued. A missing avatar can be uploaded again after login.

## Caddy route

Inside the existing private wildcard block:

```caddy
@immich host immich.example.com
handle @immich {
    reverse_proxy immich-server:2283
}
```

No new wildcard DNS record or certificate is required. The existing DNS-only wildcard resolves the hostname to VM1's Tailscale IP; the existing wildcard certificate covers it.

After Caddy access is verified, direct host publishing can be removed:

```yaml
# Remove after proxy verification:
ports:
  - "2283:2283"
```

## Verification

```bash
cd /home/services-vm/services/immich
sudo docker compose ps
sudo docker compose logs --tail=100 immich-server database
sudo docker inspect immich_server \
  --format '{{range $name,$config := .NetworkSettings.Networks}}{{$name}}{{"\n"}}{{end}}'
findmnt /mnt/storage-b
```

Operational checks:

- Previous users and albums appear after restore.
- Several old photos and videos open successfully.
- New upload succeeds and lands under the existing asset root.
- PostgreSQL files appear only under the local `postgres` directory.
- Immich remains unavailable through its private domain without Tailscale.

Keep `immich.bad-db-path-20260818` and other quarantined directories until the restored instance has a new verified backup and asset spot checks pass.
