# 28 — Build B Outage Recovery

**Scope:** storage-b/iSCSI outage affecting VM1. Do not restart broad services or write to a suspect mount.

1. On VM1, check iSCSI session, mount, kernel I/O/ext4 errors, and application state.
2. If session/mount is missing but no filesystem error exists, restore target/network, log in, mount by UUID, then test a disposable write.
3. If mount is stale/hung or kernel reports I/O/ext4 journal failure, stop writers and follow [08](08-incident-iscsi-ext4-recovery.md).
4. On storage-b, verify host/network, target service, sessions, stable backing partition, and HDD SMART. Do not mount the LUN locally while VM1 can access it.
5. Escalate to offline filesystem repair only after VM1 logout and LIO release are proven.

UPS policy: shut down VM1 writers and iSCSI cleanly before Build B battery exhaustion. A clean storage shutdown is more valuable than keeping containers alive briefly.
