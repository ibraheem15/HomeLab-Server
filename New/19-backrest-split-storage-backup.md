# 19 — Backrest Split-Storage Backup

Backrest/Restic covers VM1 local state plus Build B media mounted at /mnt/storage-b. The external repository HDD is normally offline and must be mounted/validated before Backrest starts.

## Scope

| Source | Include |
|---|---|
| VM1 local | Compose files, app config, AdGuard conf/work, Frigate config/SQLite, Immich PostgreSQL dump/strategy, keys/secrets from safe source |
| Build B iSCSI | irreplaceable photos/media; Frigate footage only if retention warrants it |

Images and packages are reproducible. Database consistency is not optional: Immich asset media without its database is an incomplete restore.

Before a backup, prove the external disk UUID/model, its mount, and existing Restic repository. A blank /repo view is a stop condition, never an instruction to initialize. Run backup, periodic restic check, and actual restore tests. Restic deduplication handles unchanged chunks across local and iSCSI source paths.

Keep Backrest configuration and repository credentials out of git. See [20](20-removable-backup-hdd-runbook.md) for physical attach/mount/removal.
