# 05 — VM1: Debian Services Guest

## VM specification

```text
VMID:        102
Name:        services-vm
Machine:     q35
Firmware:    OVMF/UEFI
CPU:         6 vCPU, type host
RAM:         24 GiB, ballooning disabled
Disk:        100 GiB on local-lvm
NIC:         VirtIO, bridge vmbr0, VLAN blank
Address:     <SERVICES_VM_LAN_IP>/24
Gateway/DNS: <LAN_GATEWAY_IP>
```

Why 100 GiB is sufficient: bulk media remains on Build B; VM1 local storage holds Debian, container images, Compose definitions, configuration, logs, and databases. Free-space monitoring remains essential because Immich PostgreSQL, thumbnails, models, and Docker layers can grow.

## Debian installation

Use Debian 13 netinst with:

- hostname `services-vm`
- SSH server
- standard system utilities
- no desktop
- single ext4 root plus EFI and swap

Static networking:

```text
auto ens18
iface ens18 inet static
        address <SERVICES_VM_LAN_IP>/24
        gateway <LAN_GATEWAY_IP>
```

The initial configuration accidentally used a second `address <LAN_GATEWAY_IP>` line instead of `gateway`, causing internet failure after reboot. A temporary `ip route add default ...` worked only until reboot; correcting `/etc/network/interfaces` made it persistent.

Validate:

```bash
ip -4 -br address
ip route
getent hosts tailscale.com
curl -I https://tailscale.com
```

## Guest services

```bash
sudo apt update
sudo apt install qemu-guest-agent openssh-server
sudo systemctl enable --now qemu-guest-agent ssh
```

The messages below appeared while enabling SSH:

```text
systemd-ssh-generator: Failed to query local AF_VSOCK CID
```

They affected optional VM-socket SSH generation, not normal TCP SSH. SSH remained usable on port 22.

Install Tailscale after default route and DNS work:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

## Accidental conversion to template

VM101 was accidentally converted into a Proxmox template. A template reserves its virtual disk but does not consume running RAM or CPU. The supported recovery used a **full clone**:

1. Select template VM101.
2. Clone → Full Clone.
3. New VMID `102`.
4. Name `services-vm`.
5. Boot and verify VM102 independently.

Do not rely on unsupported manual changes to Proxmox’s template flag. Retain template 101 until VM102, its disk, networking, Tailscale, guest agent, and applications are verified. Then delete template 101 if its disk capacity is needed.

## Docker Engine and Compose

Frigate working confirms Docker and Compose are installed. For a clean rebuild on Debian 13, use Docker’s official apt repository rather than the legacy standalone Compose binary. Current official procedure: [Install Docker Engine on Debian](https://docs.docker.com/engine/install/debian/).

Core package set:

```text
docker-ce
docker-ce-cli
containerd.io
docker-buildx-plugin
docker-compose-plugin
```

Verify:

```bash
docker --version
docker compose version
sudo systemctl is-active docker
```

If the normal user should run Docker without `sudo`:

```bash
sudo usermod -aG docker services-vm
```

Log out and back in. Membership in the `docker` group is effectively root-equivalent; grant it only to administrators.

## GPU decision

Build A has Intel UHD 630. It is not currently passed to VM1. Full iGPU passthrough could remove the Proxmox host’s local graphics console—the console that was required during the NIC outage. Frigate therefore starts with CPU OpenVINO and software video decoding. Revisit passthrough only after remote recovery paths and NIC stability are proven.

