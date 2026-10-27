# 24 — TrueNAS Datasets, ACLs, and SMB

Create child datasets, not shares of pool root:

| Dataset | Initial quota | Consumer |
|---|---:|---|
| vm2-business | 100 GiB | VM2 |
| vm3-extra | 500 GiB | VM3 |

Quotas are limits, not reservations; actual free pool capacity governs both. Keep 10–20% free.

Create separate SMB identities and ACLs. Give each VM only its own dataset, then create SMB shares for those dataset paths. Test from each VM over 10.20.0.10 before adding an automount. Keep SQLite, database, service configuration, and host keys on each VM local disk; SMB is bulk data.

After an ACL/share change, test allowed access and rejected cross-dataset access. Do not mount the SATA disk on Proxmox or share sata root. Upstream: [TrueNAS SMB shares](https://www.truenas.com/docs/scale/scaletutorials/shares/smb/).
