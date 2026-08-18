# 08 — Incident: iSCSI Timeout and Aborted ext4 Journal

## Trigger and symptoms

The Intel NIC hung long enough for VM1’s default iSCSI session recovery timeout to expire.

Timeline from VM1’s kernel log:

```text
11:50:06  iSCSI ping timeout; connection error 1022
11:52:07  session recovery timed out after 120 seconds
11:52:07  multiple read/write I/O errors on sdb
11:52:07  lost asynchronous page writes
11:52:07  ext4 journal aborted
11:52:07  ext4 attempted to remount read-only
12:03:26  another iSCSI ping timeout during network recovery
```

Application symptom:

```text
cat: .../compose.yaml: Input/output error
```

`findmnt` still displayed `rw`, but writes returned I/O errors. The kernel’s aborted-journal messages were authoritative; mount-option display alone was insufficient.

The `rg` processes shown in ext4 errors were traversing directories when the block device vanished. They exposed the failure; they did not cause it.

## Immediate rule

Stop all migration/application work. Do not run `fsck` while the LUN is mounted or exported to an active initiator.

## Step 1 — Disconnect VM1 cleanly

Inside VM1:

```bash
cd ~
sudo umount /mnt/storage-b
sudo iscsiadm -m node \
  -T <TARGET_IQN> \
  -p <STORAGE_SERVER_LAN_IP>:3260 --logout
sudo poweroff
```

If unmount reports busy:

```bash
sudo fuser -vm /mnt/storage-b
```

Stop the owning processes; do not use forced/lazy unmount for filesystem repair.

## Step 2 — Assess Build B and HDD SMART

Confirm no remote or local user:

```bash
sudo targetcli sessions
findmnt /dev/sdb2
```

SMART attributes observed:

```text
Reallocated_Sector_Ct   0
Current_Pending_Sector  0
Offline_Uncorrectable   0
UDMA_CRC_Error_Count    0
Power_On_Hours          16536
Start_Stop_Count        82393
Load_Cycle_Count        242280
Temperature             35 C
```

Interpretation: no sector-remapping, pending-sector, uncorrectable-sector, or SATA-link-error evidence. The disk is mechanically aged and heavily cycled, so the offline backup remains essential, but the simultaneous random-sector errors immediately after the session timeout implicated the network path first.

Commands:

```bash
sudo smartctl -H /dev/sdb
sudo smartctl -A /dev/sdb
```

## Step 3 — Release LIO’s exclusive device use

Even with no sessions, the active LIO block backstore keeps the partition open. `e2fsck` initially failed:

```text
... is in use.
e2fsck: Cannot continue, aborting.
exit code 8
```

Save and back up target configuration:

```bash
sudo targetcli saveconfig
sudo cp -a /etc/rtslib-fb-target/saveconfig.json \
  /root/saveconfig.pre-fsck.json
```

Clear only the live kernel target configuration:

```bash
sudo targetctl clear
sudo targetcli ls
```

`targetctl clear` releases the live backstore. It does not delete the saved JSON used for restoration. targetcli reference: [Debian targetcli manual](https://manpages.debian.org/unstable/targetcli-fb/targetcli.8.en.html).

## Step 4 — Offline ext4 repair

```bash
sudo e2fsck -f -p \
  /dev/disk/by-partuuid/<PARTITION_UUID>

echo "e2fsck exit code: $?"
```

Meaning:

```text
0  no errors
1  errors corrected
2  corrected; reboot recommended
4  errors remain
8  operational error, commonly still in use
```

`-f` forces a full check. `-p` automatically fixes only safe/unambiguous inconsistencies and stops if human judgment is required. This operation modifies filesystem metadata; the verified external backup is the fallback.

## Step 5 — Restore target and reconnect

For `e2fsck` exit `0` or `1`:

```bash
sudo targetctl restore
sudo targetcli ls
```

For exit `2`, reboot Build B; the enabled target restore service should reload the saved configuration. For exit `4+`, do not reconnect VM1—investigate or restore from backup.

Start VM1, then verify:

```bash
sudo iscsiadm -m session -P 1
findmnt /mnt/storage-b
sudo touch /mnt/storage-b/.recovery-test
sudo rm /mnt/storage-b/.recovery-test
```

The actual repair completed successfully and normal file access returned.

## Prevention added after recovery

1. Persistent NIC offload + EEE mitigation on Proxmox.
2. iSCSI `replacement_timeout` increased from 120 to 600 seconds.
3. Frigate database/config moved to VM1 local disk.
4. Remote media uses a bind mount configured not to create missing source paths.
5. Local console retained; iGPU passthrough deferred.

These controls reduce risk but do not turn single-path network block storage into redundant storage. A long outage can still require offline ext4 repair.

