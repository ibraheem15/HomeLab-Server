# 01 — Preparation and Network

Use current upstream installers: [Debian](https://www.debian.org/distrib/), [Proxmox](https://www.proxmox.com/en/downloads), and [Tailscale](https://tailscale.com/download). Keep ISO checksums, recovery credentials, and external-backup access available before disk work.

## Address plan

| Host | LAN | Storage |
|---|---:|---:|
| pve-a | 192.168.1.10 | — |
| services-vm | 192.168.1.20 | — |
| TrueNAS | 192.168.1.25 | 10.20.0.10 |
| VM2 | 192.168.1.30 | 10.20.0.20 |
| VM3 | 192.168.1.40 | 10.20.0.30 |
| storage-b | 192.168.1.51 | — |

Gateway is 192.168.1.254. Configure address, prefix, gateway, and DNS inside each OS. Test address, default route, DNS, and gateway reachability before installing software. A host with no default route may reach the LAN but cannot install packages.

## Installation media and common failures

Identify a USB by model/size, unmount its partitions, and write the ISO to the whole device. This is destructive; never guess an sdX name. If the same boot USB fails on two hosts, verify the checksum and recreate it before debugging firmware. Use Debian plain Install if the graphical installer has display issues.

Build A and Build B use ordinary Ethernet through the existing router/switch; iSCSI stays on this LAN. Do not add a VLAN tag of 1 to an untagged Proxmox guest NIC.

Install Tailscale only after IPv4 routing and DNS work. It is an administration path, never the iSCSI/SMB data path. Upstream: [Tailscale Linux install](https://tailscale.com/docs/install/linux).
