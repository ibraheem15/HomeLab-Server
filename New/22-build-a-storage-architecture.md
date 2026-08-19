# 22 — Build A Storage Architecture

## Final design

Build A contains a 500 GB-class NVMe and a 1 TB-class SATA SSD. They serve different tiers rather than being combined:

| Tier | Owner | Workload |
|---|---|---|
| NVMe | Proxmox `local-lvm` | VM operating systems, Docker state, application configuration, databases |
| SATA SSD | TrueNAS VM `101` | VM2/VM3 bulk file datasets and adjustable quotas |
| Build B HDD | VM1 through iSCSI | Immich media, Frigate recordings, Jellyfin/cold media |
| External USB HDD | Backrest/restic | Normally-off recovery copy |

The SATA disk is not divided into fixed VM partitions. TrueNAS owns the whole disk as ZFS pool `sata`; VM2 and VM3 consume file shares backed by independent datasets.

```text
Samsung SATA SSD
└── ZFS pool sata
    ├── vm2-business  quota 100 GiB
    └── vm3-faisal    quota 500 GiB
```

Quotas are limits, not reservations. They can be raised without formatting, moving data, or recreating VMs. The sum of quotas may exceed physical capacity, but actual usage cannot. Keep roughly 10–20% of the pool free.

## Why TrueNAS is appropriate here

TrueNAS was rejected for Build B because Build B had 8 GiB RAM, an existing ext4 data filesystem that could not be wiped, and only needed a lean iSCSI target. Debian + `targetcli` exported that preserved partition directly.

Build A is different: it has 48 GiB RAM, the SATA SSD could be erased, a web UI was desired, and VM2/VM3 require flexible file-level quotas. TrueNAS adds ZFS checksums, snapshots, scrubs, compression, quotas, ACLs, SMB management, SMART reporting, and alerts.

Single-disk ZFS provides detection, not redundancy. It can identify checksum damage but usually cannot repair a bad block without another copy. Snapshots on the same SSD are rollback points, not backups.

## Why SMB rather than iSCSI

TrueNAS iSCSI would expose separate block devices. Each VM would format its own zvol, resizing would require zvol + partition + filesystem changes, and two VMs could not safely mount the same ordinary filesystem.

SMB exposes datasets as files. TrueNAS owns the filesystem, applies quotas centrally, and authenticates VM2/VM3 with separate credentials. Although TrueNAS labels these as “Windows shares,” SMB is cross-platform; Linux mounts them using `cifs-utils`.

NFS would also work for Linux, but SMB was selected because username/password authentication and per-user ACL isolation are clearer than host-trusted NFS plus UID/GID coordination.

## Internal storage network

Proxmox bridge `vmbr1` is an isolated software switch with no physical port and no host IP:

```text
auto vmbr1
iface vmbr1 inet manual
        bridge-ports none
        bridge-stp off
        bridge-fd 0
# Internal VM storage network
```

Address plan:

| Node | LAN/management | Private storage |
|---|---:|---:|
| TrueNAS | `192.168.1.25/24` | `10.20.0.10/24` |
| VM2 | `192.168.1.30/24` | `10.20.0.20/24` |
| VM3 | `192.168.1.40/24` | `10.20.0.30/24` |

Only LAN interfaces receive gateway `192.168.1.254` and DNS. Storage interfaces receive no gateway and no DNS. Traffic between VMs on the same Proxmox host traverses the software bridge, not the physical 1 GbE LAN.

## Storage health and backup implications

The Samsung SATA SSD passed SMART but reported raw `Reallocated_Sector_Ct=34`. Monitor the value and TrueNAS alerts. The pool must never be the sole copy of important files.

Build B's WDC HDD reported zero reallocated, pending, and offline-uncorrectable sectors, but has high start/stop and load-cycle counts. It remains suited to capacity-oriented, sequential workloads. Immich media stays there unless measured contention justifies migration; its live PostgreSQL database stays on VM1 NVMe under all designs.
