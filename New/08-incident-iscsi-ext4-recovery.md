# 08 — Incident: iSCSI Timeout and ext4 Recovery

**Requires downtime.** Trigger: VM1 kernel reports iSCSI timeout, I/O errors, or ext4 aborted journal. Never remount read-write, fsck, or locally mount Build B’s partition until VM1 writers and LIO are proven stopped.

## Recovery order

1. Stop VM1 applications; unmount /mnt/storage-b and log out of iSCSI. Shut down VM1 if exclusivity is uncertain.
2. On Build B, verify no target sessions; inspect HDD SMART and kernel errors.
3. Save target configuration and release LIO’s claim only after proving no VM1 session exists.
4. Run an offline read-only ext4 assessment using the stable partition identifier. Approve repair mode only after the partition is unmounted and target access is gone.
5. Restore the target, log VM1 in, mount by UUID, inspect logs, and test a disposable write.
6. Start media-dependent containers only after mount and write checks pass.

Repeated I/O faults mean refresh backup and replace failing hardware; fsck is not a cure for media failure.
