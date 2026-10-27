# 10 — Operations Runbook

## Routine health check

| Host | Command | Expected |
|---|---|---|
| `pve-a` | `qm status 101 102 103; systemctl --failed; lvs` | required VMs running; no failures |
| `storage-b` | `sudo targetcli sessions; sudo targetcli ls; findmnt /dev/sdb2; sudo smartctl -H /dev/sdb` | VM1 session present; HDD not locally mounted |
| VM1 | `findmnt /mnt/storage-b; sudo iscsiadm -m session -P 1; docker ps` | one writable mount/session; expected containers |
| VM2 | `findmnt /mnt/vm2-business; cd ~/services/sftpgo && sudo docker compose ps` | SMB mount and SFTPGo present |
| proxy | `sudo ss -lntp | grep :8443; sudo tailscale serve status` | Caddy on loopback; Serve forwards `:443` |

After any storage/network fault, inspect VM1 kernel logs before a write:

```bash
sudo journalctl -k -b --no-pager | grep -Ei 'iscsi|I/O error|EXT4-fs|aborted journal' | tail -100
```

Follow [08-incident-iscsi-ext4-recovery.md](08-incident-iscsi-ext4-recovery.md) for I/O errors or an aborted journal.

## Startup and shutdown

Start: router/switch → `storage-b` and LIO → `pve-a` → TrueNAS → VM1/iSCSI mount → Docker → VM2. Verify the mount before starting any media-dependent container. VM3 remains optional until configured.

For planned shutdown: stop VM1 applications, leave the mount directory, unmount `/mnt/storage-b`, log out of iSCSI, power off VM1, then shut down Build B/Proxmox. Never locally mount or repair the LUN while VM1 is logged in.

## Backup session

1. Attach the external HDD and prove UUID, model, and expected Restic repository.
2. Quiesce or dump databases as their backup plan requires; back up VM1 configuration/databases and irreplaceable Build B media.
3. Run backup, then periodic `restic check` and an actual restore test.
4. Stop Backrest/restic, sync, unmount/eject, and disconnect the disk only after `backup-usb stop` reports `SAFE TO REMOVE`.

Images/packages are reproducible. Prioritize Compose files, application config, databases, keys, and personal media.

## Maintenance policy

- Before network, fstab, target, or Compose edits, copy the exact file with a dated `.before-*` suffix.
- Monthly: SMART health/attributes on Build B; inspect TrueNAS pool alerts and free space; check Proxmox thin-pool use.
- Upgrade OS, Proxmox, Tailscale, and Docker routinely. Read release/migration notes and back up before major app or kernel upgrades.
- Perform disruptive reboot/failure tests only in a maintenance window and record the result in `00`.
