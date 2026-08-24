# Home Lab Wiki — Agent Instructions

Everything inside "New" is wiki content. Everything inside "Old" is a historical record of the build(Archived). The wiki is a living document — update it when you change the build, and verify that the instructions still work.

**Mission:** Maintain/extend this build safely. This dir is both reconstruction guide and operational record. Correct + reversible > fast. Operator is technically capable but wants one verified step at a time — don't dump a full migration unless asked. Each step: exact host, file, command, expected result, rollback boundary. Wait for result before continuing.

## Read before changing anything
1. `README.md` — topology, wiki map, invariants
2. `00-current-state.md` — live inventory, unfinished work (authoritative)
3. Numbered doc for the affected component
4. `11-troubleshooting-index.md` — known symptom→fix
5. `10-operations-runbook.md` — reboot/outage/backup/maintenance

Never infer state from an old build guide alone — verify live, update `00-current-state.md`, flag stale instructions.

## Topology

| System | Role | LAN IP | Storage-net IP |
|---|---|---|---|
| pve-a | Proxmox VE host | 192.168.1.10 | bridge only |
| TrueNAS VM 101 | ZFS/SMB for VM2+VM3 | 192.168.1.25 | 10.20.0.10 |
| services-vm (VM1) | Docker services | 192.168.1.20 | — |
| drive-vm (VM2) | SFTPGo | 192.168.1.30 | 10.20.0.20 |
| owner-vm (VM3) | co-owner | 192.168.1.40 | 10.20.0.30 (per current-state) |
| storage-b | Debian storage target | 192.168.1.51 | — |

Gateway: `192.168.1.254`. Static IPs set **inside each OS** — never router DHCP/port-forward. Tailscale = private remote access. Cloudflare Tunnel = explicit public exceptions only.

## Storage ownership

| Source | → | Owner |
|---|---|---|
| Build A 500GB NVMe | → | Proxmox + VM OS disks |
| Build A 1TB SATA SSD | → | passed through whole to TrueNAS VM 101, pool `sata` |
| `sata/vm2-business`, `sata/vm3-faisal` | → | VM2/VM3 bulk data |
| Build B HDD partition | → | iSCSI, exclusively to VM1, mounted `/mnt/storage-b` |
| External Restic/Backrest HDD | → | normally off; mount only for backup sessions |

**DB rule:** Postgres/SQLite/container config/mutable app state → local NVMe-backed disk on the relevant VM, always. Network storage = bulk media/files only, unless a component guide explicitly overrides this.

## Hard rules (never break without explicit confirmation)

- Never wipe/format/repartition/init/import/overwrite a disk without confirming the stable device ID + explicit data-loss acknowledgment.
- Never use `/dev/sdX` as identity — resolve `/dev/disk/by-id`, UUID, PARTUUID, model, serial, fs, mounts, consumers first.
- Never mount Build B's exported ext4 locally while VM1 holds an iSCSI session (not a clustered fs).
- Never `fsck` a mounted fs or online LUN — stop writers → unmount → end session/export → prove exclusivity → repair.
- Never mount/format TrueNAS's 1TB SATA SSD from Proxmox. Never share the root ZFS pool — use child datasets.
- Never start a container against an absent network mount — preserve fail-closed binds (`create_host_path: false`), verify via `findmnt`/`mountpoint`.
- Never put app DBs on iSCSI/SMB/NFS/removable storage unless user explicitly changes architecture after reviewing corruption/recovery risk.
- Never expose an admin service publicly by accident — keep LAN, Tailscale-private, Cloudflare-public routes distinct.
- Never print/commit passwords, CHAP secrets, tokens, keys, camera creds, `.env` contents — redact shown secrets, recommend rotation.
- Never touch unrelated working services while fixing one thing. No broad Docker/Proxmox/network restarts.

## Working method

**1. Evidence first.** Read-only checks: `hostnamectl`, `ip -br address`, `ip route`, `lsblk`, `findmnt`, `systemctl status`, `journalctl`, `docker compose ps`, `docker inspect`, component-specific diagnostics. State likely cause separately from confirmed fact. If the error was in a prior boot, inspect that boot — current health ≠ proof of past cause.

**2. Plan.** Nail down: host/layer (Proxmox host vs VM vs container), data path/ownership, dependencies + boot order, expected interruption, verification command, rollback path. Ask before destructive/public/credential/architecture changes. Reversible in-scope config changes may proceed once evidence is sufficient.

**3. Execute one checkpoint.** Copyable command blocks, every placeholder labeled. Prefer stable names/UUIDs/systemd units/Compose dirs, idempotent commands. After each checkpoint, ask only for the output needed for the next decision — don't repeat completed steps.

**4. Verify both directions.** Normal operation ≠ proof. Where relevant, test: service/VM reboot, full Proxmox reboot, dependency starts late, dependency vanishes mid-run, mount absent, bad creds/ACL rejected, backup restorable. No disruptive resilience tests without a maintenance window + explicit approval.

## Docker / ingress conventions

- Compose projects under each VM user's `services/` — confirm actual path before editing.
- Shared reverse-proxy network: external Docker network `edge`. Only web-facing containers join `edge`; DBs/caches stay on Compose-private networks.
- Caddy proxies to container DNS names/ports on `edge`, not published host ports (published ports OK for LAN/Tailscale diagnostics).
- Cloudflare Tunnel = HTTP apps only, unless a documented TCP-access design exists. Native SFTP → drive-vm LAN/Tailscale IP, port 2022 — not the tunnel hostname.
- Recreate (not restart) after Compose/env changes.
- Keep explicit health checks + fail-closed storage guards.

## Boot order

- Storage-dependent services wait for the **actual mount**, not `network-online.target`. Docker restart policy handles a process that exits after starting — not one rejected pre-start from an absent bind source.
- Use per-stack systemd orchestration for storage-dependent Compose projects. Don't gate the whole Docker daemon on Build B/TrueNAS.
- Frigate: keep the systemd guard verifying `/mnt/storage-b`, retrying until iSCSI is up. NFS migration stays deferred until its own runbook is run intentionally.

## Performance diagnosis

Measure each segment before redesigning anything: `client → Wi-Fi/LAN/Tailscale → app VM → storage protocol → storage VM/disk`. Use large-file transfer for throughput, then compare small-file behavior. Baseline: VM-to-VM SFTP ≈ **241.8 MB/s** (SFTPGo→SMB→TrueNAS path healthy) — so Android slowness points to Wi-Fi/routing/client, not this path.

## After a confirmed change

1. Update `00-current-state.md` with resulting live state (not intended state).
2. Update the relevant numbered guide: commands, error hit, root cause, fix, verification, rollback.
3. New numbered doc only if it doesn't fit an existing one — next unused two-digit prefix.
4. Update `11-troubleshooting-index.md` for reusable symptom→cause→fix.
5. Update `README.md` if topology/scope/safety/wiki map changed.
6. Never silently rewrite history — record corrections explicitly. Placeholders for secrets, keep real non-secret identifiers needed for recovery.

## Response format

Lead with diagnosis/required action. Compressed but causal. Tables for exact mappings. Label clearly: **safe now** / **requires downtime** / **destructive**. Guided execution → stop after one actionable checkpoint unless full runbook requested. Completed task → report what changed, what was verified, which wiki files updated.

## Definition of done

Intended state verified from the correct host · reboot/persistence verified where relevant · no unintended public exposure or local fallback dir · backup/rollback implications known · wiki reflects confirmed result · no secrets in committed docs.