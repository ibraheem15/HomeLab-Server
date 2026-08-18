# 03 — Build B: iSCSI Target with ACL and CHAP

## Target definition

```text
Target IQN:    <TARGET_IQN>
Portal:        <STORAGE_SERVER_LAN_IP>:3260
Backstore:     media-hdd
Backing block: /dev/disk/by-partuuid/<PARTITION_UUID>
Initiator IQN: <INITIATOR_IQN>
CHAP user:     <CHAP_USERNAME>
CHAP password: <SECRET>
```

The target exports the existing ext4 partition as a raw block device. VM1 mounts the filesystem; Build B does not.

## Install and enable targetcli

```bash
sudo apt update
sudo apt install targetcli-fb
sudo systemctl enable --now rtslib-fb-targetctl.service
```

Verify:

```bash
dpkg-query -W targetcli-fb
systemctl is-enabled rtslib-fb-targetctl.service
```

## Create the target

Use interactive `targetcli`; exit with `exit` or `Ctrl+D`. Never use `Ctrl+Z`, which suspends the process and can leave multiple stopped shells.

```bash
sudo targetcli
```

Representative configuration:

```text
/backstores/block create name=media-hdd dev=/dev/disk/by-partuuid/<PARTITION_UUID> readonly=false
/iscsi create <TARGET_IQN>
/iscsi/<TARGET_IQN>/tpg1/portals create <STORAGE_SERVER_LAN_IP> 3260
/iscsi/<TARGET_IQN>/tpg1/luns create /backstores/block/media-hdd
/iscsi/<TARGET_IQN>/tpg1 set attribute generate_node_acls=0 authentication=1
/iscsi/<TARGET_IQN>/tpg1/acls create <INITIATOR_IQN>
/iscsi/<TARGET_IQN>/tpg1/acls/<INITIATOR_IQN> set auth userid=<CHAP_USERNAME> password=<SECRET>
saveconfig
exit
```

If targetcli automatically created a wildcard portal such as `0.0.0.0:3260`, delete it and retain only `<STORAGE_SERVER_LAN_IP>:3260`.

Inspect rather than assuming object names:

```bash
sudo targetcli ls
sudo targetcli sessions
```

Expected tree properties:

- Backstore `media-hdd` points to the PARTUUID path.
- Backstore `readonly: False`.
- One LUN under `tpg1`.
- One ACL for VM1’s exact IQN.
- Mapped LUN is read/write.
- Authentication is enabled.
- Portal listens on Build B’s LAN address.

## Write-protection troubleshooting

Symptom in VM1:

```text
WARNING: source write-protected, mounted read-only
```

Check both layers on Build B:

```bash
sudo targetcli /backstores/block/media-hdd info
sudo blockdev --getro \
  /dev/disk/by-partuuid/<PARTITION_UUID>
```

Expected:

```text
readonly: False
0
```

If the ACL’s mapped LUN is read-only, inspect it with `targetcli ls`, then set its `write_protect` property to `0` and reconnect the initiator. A target-side change is not necessarily visible until the initiator logs out and back in.

## targetcli appeared stuck

Cause: `Ctrl+Z` suspended multiple targetcli processes:

```text
[1] Stopped sudo targetcli
[2] Stopped sudo targetcli
...
```

Recovery:

```bash
jobs -l
kill %1 %2 %3 %4
jobs
ps -ef | grep '[t]argetcli'
```

Use non-interactive commands with a timeout while diagnosing:

```bash
sudo timeout 15 targetcli /backstores/block/media-hdd info
```

## Persistence test

After saving:

```bash
sudo targetcli saveconfig
sudo reboot
```

Then verify:

```bash
sudo targetcli ls
sudo ss -lntp | grep 3260
```

The saved configuration is stored in:

```text
/etc/rtslib-fb-target/saveconfig.json
```

This file contains CHAP credentials in cleartext and must remain root-readable only.

