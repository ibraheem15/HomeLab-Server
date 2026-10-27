# 11 — Troubleshooting Index

| Symptom | Likely cause | Safe first action |
|---|---|---|
| VM1 mount exists but writes return I/O errors | iSCSI loss / ext4 aborted journal | stop writers; inspect kernel log; use [08](08-incident-iscsi-ext4-recovery.md) |
| `e2fsck` says device in use | LIO still owns Build B disk | prove VM1 logout; clear/stop target before repair |
| LUN changes `sdb`/`sdc` | normal SCSI enumeration | use UUID/PARTUUID, never device letters |
| iSCSI login rejected | ACL/IQN or CHAP mismatch | compare initiator IQN and secret without printing it |
| Caddy private hostname times out | missing Tailscale/Serve/DNS path | `tailscale serve status`, loopback `:8443`, then DNS |
| Caddy returns 502 | backend not reachable on `edge` | inspect container network/name/port; recreate after Compose change |
| public hostname hits private service | exact Cloudflare record missing/stale | preserve DNS-only wildcard; fix only explicit record |
| Frigate stores media on VM1 root | storage mount absent/wrong bind | stop container; restore mount; retain `create_host_path: false` |
| AdGuard cannot bind `:53` | old resolver still owns port | stop Pi-hole/Technitium; set restart policy `no`; prove port free |
| AdGuard clients all show `172.x` | bridge NAT | use documented host network mode |
| Immich DB DNS failure | PostgreSQL unhealthy or networks split | fix DB first; ensure both join `immich-internal` |
| Backrest offers to initialize | USB not mounted/incorrect mount | stop; verify UUID and existing repo; never initialize blindly |
| VM2 SMB is absent | storage network/CIFS mount issue | test `ping 10.20.0.10`, then manual mount before Docker |

## Diagnostic bundles

```bash
# VM1 storage
findmnt /mnt/storage-b
sudo iscsiadm -m session -P 1
sudo journalctl -k -b --no-pager | grep -Ei 'iscsi|I/O error|EXT4-fs' | tail -150

# Build B target
sudo targetcli sessions
sudo targetcli ls
findmnt /dev/sdb2
sudo smartctl -H /dev/sdb

# VM1 proxy
cd /home/services-vm/services/proxy
sudo docker compose ps
sudo docker compose logs --tail=100 caddy cloudflared
sudo ss -lntp | grep :8443
sudo tailscale serve status
```

Redact credentials and tokens before sharing output. See the linked component guide for a repair procedure; this file deliberately stops at safe diagnosis.
