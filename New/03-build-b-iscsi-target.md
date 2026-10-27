# 03 — Build B: iSCSI Target

Export the preserved ext4 partition to VM1 only. Follow current [targetcli/LIO documentation](https://github.com/open-iscsi/targetcli-fb) for release-specific syntax.

## Required configuration

| Item | Value |
|---|---|
| Portal | storage-b LAN address, TCP 3260 |
| Backstore | stable HDD partition PARTUUID path |
| Target | documented target IQN |
| ACL | VM1’s exact initiator IQN |
| Authentication | CHAP required |
| Filesystem owner | VM1 only |

Install targetcli-fb and enable its restore service. In targetcli, create one read/write block backstore, target, LAN-only portal, LUN, VM1 ACL, and CHAP credentials; then save configuration. Delete any wildcard portal. Store secrets outside this wiki.

Verify after every change and reboot: target tree has one correct backstore/LUN/ACL, authentication is enabled, the target listens only on its LAN address, and a session appears only when VM1 is connected. The saved target file contains CHAP credentials and stays root-readable only.

## Failure notes

A read-only VM1 LUN can be caused by the backstore or ACL mapped-LUN flag; inspect both, correct it, then reconnect VM1. Do not suspend targetcli with Ctrl-Z; exit normally. See [06](06-vm1-iscsi-initiator.md) for the initiator and [08](08-incident-iscsi-ext4-recovery.md) for failure recovery.
