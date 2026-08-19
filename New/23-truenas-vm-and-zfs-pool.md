# 23 — TrueNAS VM and ZFS Pool

## VM definition

| Setting | Value |
|---|---|
| VMID/name | `101` / `truenas` |
| Release | TrueNAS Community Edition 25.10.6 |
| Machine/firmware | Q35 + OVMF/UEFI; Secure Boot keys disabled |
| CPU | 2 cores, type `host` |
| RAM | 8192 MiB fixed; ballooning disabled |
| Boot disk | 32 GiB SCSI on `local-lvm`; discard + SSD emulation |
| `net0` | VirtIO on `vmbr0` |
| `net1` | VirtIO on `vmbr1` |
| Data disk | Whole Samsung 870 EVO by stable `/dev/disk/by-id` path |

Raw disk attachment on Proxmox:

```bash
qm set 101 -scsi1 \
  /dev/disk/by-id/ata-Samsung_SSD_870_EVO_1TB_S626NJ0R325183M,backup=0,cache=none,discard=on,iothread=1,ssd=1
```

The VM must be stopped for storage-definition changes. The data disk is never mounted by Proxmox and is excluded from VM backup because the VM configuration references a physical device, not a Proxmox volume.

## Installation incidents

### Corrupted installer display

The noVNC framebuffer showed unreadable colored lines. Changing the VM console/display to SPICE fixed the installer. This was display emulation only; disk contents were unaffected.

During installation, only the 32 GiB QEMU boot disk was selected. The 931.5 GiB Samsung disk was deliberately excluded. Authentication method **Administrative user (`truenas_admin`)** created the Web UI administrator during installation.

After installation, the ISO was removed and `scsi0` placed first in boot order.

### Duplicate serial error

Pool creation initially failed:

```text
Error: topology
Disks have duplicate serial numbers: None (sda, sdb)
```

Both emulated SCSI devices lacked reported serials. The fix preserved each existing disk definition and appended unique serials:

```bash
qm shutdown 101

BOOT_CFG=$(qm config 101 | sed -n 's/^scsi0: //p')
DATA_CFG=$(qm config 101 | sed -n 's/^scsi1: //p')

qm set 101 -scsi0 "${BOOT_CFG},serial=TRUENASBOOT32"
qm set 101 -scsi1 "${DATA_CFG},serial=S626NJ0R325183M"

qm config 101 | grep '^scsi'
qm start 101
```

## Networking

TrueNAS initially showed only one interface because the second VirtIO adapter required verification/full power cycle. Final addresses:

```text
Management interface: 192.168.1.25/24
Storage interface:    10.20.0.10/24
Default gateway:      192.168.1.254
DNS:                  192.168.1.254 plus public fallback
```

The storage interface has no gateway. The Web UI is reached through `http://192.168.1.25`; SMB clients use `10.20.0.10`.

## Pool creation

Creating pool `sata` permanently erased the previous NTFS partition after explicit confirmation.

Pool settings:

```text
Name: sata
Data VDEV: one-disk Stripe
Disk: approximately 931.5 GiB Samsung SATA SSD
Encryption: disabled
Deduplication: disabled
Compression: inherited/default LZ4
Log/cache/spare/special VDEVs: none
```

“Stripe” is the ZFS layout label for a single data disk. It provides no redundancy. Do not store ordinary files in the pool root; create child datasets.

## Startup order

Recommended Proxmox order:

```text
1  TrueNAS VM 101
2  services-vm 102
3  VM2 103
4  VM3 104
```

Give TrueNAS approximately 30 seconds before SMB-dependent VMs start. On shutdown, stop SMB consumers before TrueNAS.
