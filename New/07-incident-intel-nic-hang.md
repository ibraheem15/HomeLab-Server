# 07 — Incident: Intel e1000e NIC Hang

**Impact:** pve-a NIC watchdog/hardware-unit hangs interrupted VM1 iSCSI and caused an ext4 aborted journal. Treat this as a host/network event with storage consequences.

## Response

1. Inspect logs from the boot that failed for e1000e, watchdog, reset, and link events.
2. Restore link; do not write through a suspect VM1 mount.
3. If VM1 reports I/O errors or ext4 journal abort, follow [08](08-incident-iscsi-ext4-recovery.md).
4. Preserve evidence before resetting or upgrading unrelated components.

The persistent host driver/module mitigation is documented in the host configuration; back it up before changing it and verify it after a reboot. Schedule NIC firmware/BIOS updates separately. Recurrence requires checking driver, firmware, cable, switch port, and kernel logs together.
