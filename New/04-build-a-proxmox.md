# 04 — Build A: Proxmox VE Installation and Storage

## Physical disks

Build A contains two SSDs:

```text
nvme0n1  476.9G  Samsung NVMe
sda      931.5G  Samsung SATA SSD, NTFS
```

Proxmox was installed on the 500 GB NVMe. The 1 TB SATA SSD was left untouched for later use. This choice keeps the faster NVMe for hypervisor metadata, VM operating systems, and databases while preserving flexibility for the larger disk.

## Installation

1. Boot the corrected Proxmox USB in UEFI mode.
2. Select the 500 GB NVMe by model and capacity.
3. Hostname: `pve-a`.
4. Address: `<PVE_LAN_IP>/24`.
5. Gateway/DNS: `<LAN_GATEWAY_IP>`.
6. Complete installation and remove USB.
7. Open `https://<PVE_LAN_IP>:8006`.

Post-install verification:

```bash
hostnamectl
ip -4 -br address
ip route
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,MODEL
pvs
vgs
lvs
pveversion -v
```

Observed Proxmox release:

```text
proxmox-ve 9.2.0
pve-manager 9.2.2
kernel 7.0.2-6-pve
```

## Actual Proxmox storage allocation

```text
pve-root       64 GiB
pve-swap        8 GiB
pve-data      346.9 GiB thin pool
VG unallocated 50 GiB
```

The original 1.5 TB single-SSD assumption was wrong twice:

1. The hardware is two drives: 500 GB NVMe + 1 TB SATA SSD.
2. Decimal TB/GB labels are larger than binary TiB/GiB values shown by Linux.

VM1 received a 100 GiB thin-provisioned disk on the NVMe. Future VM disks can be added from unused thin-pool/VG capacity or from the 1 TB SSD after intentionally converting it to Proxmox storage. Existing VM disks do not require reformatting when adding another virtual disk.

## Repository and updates

Configure the supported repository appropriate to the subscription state. For a non-subscription home lab, use Proxmox’s no-subscription repository and remove/disable enterprise repository entries that require credentials. Then:

```bash
apt update
apt full-upgrade
pveversion -v
```

Repository syntax changes across Proxmox/Debian releases; follow the installed version’s official package-repository documentation rather than copying an older `sources.list` snippet.

## Tailscale on Proxmox

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
tailscale status
```

Tailscale is a recovery/administration path. Preserve local console access because network-driver failures can take down both LAN and Tailscale simultaneously.

## Deliberate iSCSI exclusion

Do not install or configure `open-iscsi` on Proxmox. If the host logged into the LUN, Proxmox could see or accidentally mount the same filesystem intended exclusively for VM1. VM1’s initiator identity + CHAP enforce the boundary.

## Future VM storage

Creating a new 200 GiB VM does not format the entire SSD. Proxmox creates a new logical/virtual disk inside an existing storage pool. Before allocation:

```bash
pvesm status
lvs
vgs
```

Thin provisioning permits allocated virtual sizes to exceed currently consumed physical blocks, but physical exhaustion is catastrophic. Maintain meaningful free capacity and monitor the thin pool’s `Data%` and `Meta%`.

The untouched NTFS disk must not be added to `local-lvm` casually. Decide whether to preserve its contents, back them up and reformat it as LVM/ZFS/directory storage, or pass it through to one VM.

