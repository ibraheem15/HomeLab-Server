# 28 — Build B Outage Recovery

## Scope

Use this runbook when Build B (`storage-b`) stops or reboots while Build A and VM1 (`services-vm`, VMID `102`) remain running. This is the difficult failure mode because VM1 may retain a mounted ext4 filesystem after the backing iSCSI disk disappears.

After Build B returns, several outcomes are possible:

1. iSCSI reconnects automatically and ext4 resumes normally.
2. The mount returns, but storage-dependent containers remain unhealthy.
3. The block device or mount remains stale or disappears.
4. ext4 aborts its journal or remounts the filesystem read-only.

Do not assume that an existing mount is healthy. Verify the iSCSI session, mount mode, kernel log, and a real write before restarting workloads.

## Initial checks inside VM1

Confirm that Build B's iSCSI port is reachable and that the session, block device, and mount have returned:

```bash
nc -vz 192.168.1.51 3260
sudo iscsiadm -m session
lsblk -f
findmnt /mnt/storage-b
findmnt -no OPTIONS /mnt/storage-b
```

Check the current boot's kernel log for storage failures:

```bash
sudo journalctl -k -b --no-pager \
  | grep -Ei 'iscsi|I/O error|Buffer I/O|EXT4-fs error|aborted journal' \
  | tail -100
```

If the mount exists, reports `rw`, and the kernel log has no aborted-journal or I/O errors, test an actual write:

```bash
sudo touch /mnt/storage-b/.recovery-test
sudo rm /mnt/storage-b/.recovery-test
```

An `rw` option in `findmnt` is not sufficient by itself. If either command returns `Input/output error` or `Read-only file system`, stop storage-dependent services and follow [08 — Incident: iSCSI Timeout and Aborted ext4 Journal](08-incident-iscsi-ext4-recovery.md). Never run `fsck` against the mounted or actively exported LUN.

If the checks and write test pass, restart the affected applications:

```bash
cd /home/services-vm/services/immich
sudo docker compose restart

cd /home/services-vm/services/frigate
sudo docker compose restart
```

Then confirm container health and review recent logs:

```bash
sudo docker ps

cd /home/services-vm/services/immich
sudo docker compose ps
sudo docker compose logs --tail=100

cd /home/services-vm/services/frigate
sudo docker compose ps
sudo docker compose logs --tail=100
```

## If iSCSI did not recover

If VM1 has the readiness service described for this deployment, restart it and recheck the mount:

```bash
sudo systemctl restart storage-b-ready.service
systemctl status storage-b-ready.service --no-pager
findmnt /mnt/storage-b
```

If `storage-b-ready.service` has not been created, log in manually using the saved node configuration:

```bash
sudo iscsiadm -m node \
  -T iqn.2026-08.arpa.home:storage-b.media \
  -p 192.168.1.51:3260 \
  --login

sudo udevadm settle
sudo mount /mnt/storage-b
```

Re-run all initial checks, including the kernel-log and write tests, before restarting containers. If `iscsiadm` reports that no matching node record exists, restore the initiator configuration and CHAP settings described in [06 — VM1 iSCSI Initiator](06-vm1-iscsi-initiator.md); do not place the CHAP secret directly in shell history.

## If VM1 has a stale or hung mount

Stop storage-dependent containers before attempting a normal unmount. Do not force-unmount a filesystem with active writes.

Inside VM1:

```bash
cd /
sudo docker stop frigate immich_server immich_machine_learning
sudo fuser -vm /mnt/storage-b
sudo umount /mnt/storage-b
```

Stop any additional process reported by `fuser`, then retry the normal unmount. Do not use lazy or forced unmount as a shortcut before filesystem repair.

If storage commands hang in uninterruptible I/O, control VM1 from the Build A Proxmox shell:

```bash
qm reboot 102
```

If the graceful reboot fails, hard-stop and start VM1:

```bash
qm stop 102
qm start 102
```

A hard stop may lose in-flight VM1 writes, so it is an escalation step rather than the first response. Build A itself does not need to be rebooted solely because VM1's iSCSI mount is stale.

After VM1 returns, repeat the initial checks. If ext4 is read-only, reports I/O errors, or has an aborted journal, keep applications stopped and perform the offline recovery in [08-incident-iscsi-ext4-recovery.md](08-incident-iscsi-ext4-recovery.md).

## Failure expectations

| Failure condition | Expected recovery behavior |
|---|---|
| Both hosts abruptly lose power | The systems should recover automatically when started in the documented dependency order |
| VM1 starts before Build B | The readiness service should wait and retry until Build B becomes available |
| Build B restarts and returns within 10 minutes | Open-iSCSI should reconnect within the configured 600-second replacement timeout |
| Containers fail temporarily during the outage | Docker restart policies should recover them after storage becomes healthy |
| Mount returns read-only | Manual recovery is required; do not restart storage-dependent containers |
| iSCSI device becomes stale | Restart VM1, not Build A |
| ext4 reports corruption or an aborted journal | Perform an offline `e2fsck` using the documented recovery procedure |
| Physical HDD fails | Replace or repair the disk and restore the affected data from backup |

Automatic recovery is an expectation, not proof that the filesystem is healthy. Always complete the initial checks and write test before considering the incident resolved.

## Recovery escalation

| Condition | Required action |
|---|---|
| Mount returns `rw`, the kernel log is clean, and the write test passes | Restart affected containers and verify their health |
| iSCSI session is missing | Restart the readiness service or log in manually |
| Mount is read-only or ext4 reports an aborted journal | Stop services, disconnect the LUN safely, and perform an offline filesystem check |
| Mount is stale or storage commands hang | Restart VM1 |
| VM1 cannot be controlled gracefully | Hard-stop and start VM1 from Proxmox |
| Proxmox networking or Build A's NVMe has failed | Diagnose or restart Build A in a maintenance window |
| Build B's target service or HDD has failed | Repair Build B; restarting Build A will not restore the storage |

## UPS shutdown policy

Configure the UPS to begin a graceful shutdown early enough to finish before battery exhaustion. Preserve the storage dependency order:

```text
UPS reaches low-battery threshold
  -> stop VM1 storage-dependent services
  -> shut down VM1 cleanly
  -> shut down the remaining Build A VMs
  -> shut down Build A
  -> shut down Build B last
```

Build B remains available until VM1 has stopped writing and disconnected. On restoration, use the reverse dependency order: network first, Build B and its iSCSI target next, Build A after that, and VM1 workloads only after storage validation succeeds.
