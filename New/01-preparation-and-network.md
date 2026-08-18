# 01 — Preparation, Installation Media, and Network Plan

## Phase 0 — Downloads and credentials

Prepare before changing any disks:

- Proxmox VE ISO
- Debian 13 netinst ISO
- Two USB drives
- A USB writer such as Rufus/Etcher, or Linux `dd`
- Current offline Backrest/restic repository and credentials
- Tailscale login
- Later phases: Cloudflare and registrar credentials

Verify downloaded image checksums against the publisher before writing USB media.

### Writing a Debian USB from Arch Linux

Identify the USB carefully:

```bash
lsblk -o NAME,SIZE,MODEL,TRAN,MOUNTPOINTS
```

Unmount its mounted partitions, then write the ISO to the whole device—not a partition:

```bash
sudo umount /dev/sdX* 2>/dev/null
sudo dd if=/path/to/debian.iso of=/dev/sdX bs=4M status=progress oflag=sync
sync
```

`/dev/sdX` is destructive. Resolve it from `lsblk`; never guess, and never use `/dev/sdX1`.

## Phase 1 — Static addressing

The LAN is `<LAN_CIDR>`; gateway and DNS resolver are `<LAN_GATEWAY_IP>`. Addresses are configured at the OS layer:

```text
pve-a       <PVE_LAN_IP>/24
services-vm <SERVICES_VM_LAN_IP>/24
drive-vm    <DRIVE_VM_LAN_IP>/24  (future)
owner-vm    <OWNER_VM_LAN_IP>/24  (future)
storage-b   <STORAGE_SERVER_LAN_IP>/24
```

The initial Build B plan used `.50`. `.51` was chosen during installation to distinguish the new SSD-based Debian installation from the old HDD boot while migrating.

**Required fields for every static configuration:** address/prefix, gateway, and DNS. An address without `gateway` creates only the connected-subnet route; LAN access works, but internet downloads fail.

Validate every host:

```bash
ip -4 -br address
ip route
getent hosts debian.org
ping -c 3 <LAN_GATEWAY_IP>
```

Expected route:

```text
default via <LAN_GATEWAY_IP> dev <interface>
```

## Phase 2 — Physical wiring

Connect Build A and Build B by Ethernet to the existing router/switch. iSCSI is carried over this LAN. Verify link/activity LEDs before diagnosing software.

## Boot-menu and USB lessons

### USB initially not detected

The first Proxmox USB image was not recognized on either machine; recreating the image fixed it. Cross-testing the same USB on Build A and Build B isolated the problem to installation media rather than Lenovo USB ports.

General sequence:

1. Re-download or checksum-verify the ISO.
2. Rewrite the USB in raw/image mode.
3. Use a rear motherboard USB port.
4. Open the one-time boot menu and select the UEFI USB entry.
5. If the same USB fails on two computers, treat the image/write as the primary suspect.

### Graphical installer black screen

“Graphical install” changes only the installer UI; it does not install a desktop unless desktop tasks are selected later. On server hardware with display-mode problems, select plain **Install** from the Debian boot menu. The text installer supports the same disk, network, SSH, and package choices.

### Server partitioning choice

Separate `/var`, `/tmp`, and `/home` partitions are possible but unnecessary on the 120 GB Build B SSD. A single ext4 root plus swap was selected to avoid fixed-size partitions becoming full. Service data is not stored on this SSD.

## Network failure patterns encountered

### DNS resolved only IPv6; `curl` could not connect

Symptom:

```text
getent hosts tailscale.com     # returned IPv6 addresses
curl ...                       # could not connect to port 443
ip -4 route                    # no default route
```

Temporary repair:

```bash
sudo ip route add default via <LAN_GATEWAY_IP> dev ens18
```

Permanent repair in `/etc/network/interfaces`:

```text
auto ens18
iface ens18 inet static
        address <SERVICES_VM_LAN_IP>/24
        gateway <LAN_GATEWAY_IP>
```

The failed configuration mistakenly contained a second `address <LAN_GATEWAY_IP>`; a gateway is not a second host address.

### Debian installer “Bad archive mirror”

VM1’s Proxmox NIC had VLAN tag `1` even though the LAN is untagged. The guest could not reach the Debian mirror. Clear the VLAN field; do not use `1` as a synonym for “default.” Correct NIC settings:

```text
Bridge: vmbr0
Model: VirtIO
VLAN Tag: blank
Firewall: optional; initially off for installation diagnosis
```

## Tailscale baseline

Install after basic IPv4 routing and DNS work:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale status
```

Linux runs Tailscale as a system service, so it remains available without a logged-in user. Official instructions: [Install Tailscale on Linux](https://tailscale.com/docs/install/linux).

Tailscale does not replace correct LAN routing. iSCSI remains on `<SERVICES_VM_LAN_IP> ↔ <STORAGE_SERVER_LAN_IP>`; Tailscale is for administration.

