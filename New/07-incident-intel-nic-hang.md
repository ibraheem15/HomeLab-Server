# 07 — Incident: Intel `e1000e` Hardware Unit Hang

## Impact

Build A simultaneously lost:

- Proxmox UI/SSH at `<PVE_LAN_IP>`
- Tailscale access to `pve-a`
- VM1 LAN/Tailscale access at `<SERVICES_VM_LAN_IP>`
- VM1’s iSCSI path to Build B

Because both the host and guest disappeared together, the initial fault domain was Build A power/network—not VM1 networking.

## Evidence

At the local Proxmox console:

```text
vmbr0 UP <PVE_LAN_IP>/24
default via <LAN_GATEWAY_IP> dev vmbr0
0 failed systemd units
ping 1.1.1.1 -> Destination Host Unreachable
```

The bridge configuration and route were intact. Kernel logs repeated every two seconds:

```text
e1000e 0000:00:1f.6 nic0: Detected Hardware Unit Hang
```

Interpretation: `vmbr0` remained logically up, but the Intel physical NIC’s transmit hardware stopped making progress. Since Tailscale also depends on the physical NIC, it could not provide an alternate path.

## Immediate recovery

At Build A’s local console:

```bash
ethtool -K nic0 tso off gso off gro off
ethtool --set-eee nic0 eee off
ip link set nic0 down
sleep 2
ip link set nic0 up
ifreload -a
```

Validate:

```bash
ping -c 4 <LAN_GATEWAY_IP>
ip -br link
ping -c 4 <SERVICES_VM_LAN_IP>
```

This recovered connectivity without rebooting the host.

## What the settings mean

```text
TSO — NIC segments large TCP buffers into packets
GSO — kernel defers segmentation of oversized packets
GRO — kernel combines received packets
EEE — Energy Efficient Ethernet low-power idle transitions
```

Offloads reduce CPU work. EEE reduces idle power. The recovery changed all four features and reset the link, so a single exact trigger was not proven. The observed failure is consistent with an interaction among the Intel I219-family controller, `e1000e`, offloads/EEE, bridging, and sustained network traffic.

## Persistent mitigation

In `/etc/network/interfaces`, the physical interface has no IP; it is a bridge member. Add post-up actions to its existing stanza:

```text
iface nic0 inet manual
        post-up /usr/sbin/ethtool -K nic0 tso off gso off gro off
        post-up /usr/sbin/ethtool --set-eee nic0 eee off
```

Do not move Proxmox’s address from `vmbr0` to `nic0`. The final bridge remains:

```text
auto vmbr0
iface vmbr0 inet static
        address <PVE_LAN_IP>/24
        gateway <LAN_GATEWAY_IP>
        bridge-ports nic0
        bridge-stp off
        bridge-fd 0
```

Verify current runtime state:

```bash
ethtool -k nic0 | grep -E 'tcp-segmentation|generic-segmentation|generic-receive'
ethtool --show-eee nic0
```

Expected relevant values: `off`; EEE disabled.

Tradeoff: slightly higher host CPU and power usage, and potentially lower peak throughput. On a 1 GbE link with an i7-10700T, stability is more important.

## Longer-term maintenance

The installed Lenovo firmware was dated 2023. Lenovo lists newer M70q firmware; update during a controlled window with:

1. Verified current backups.
2. VM1 cleanly stopped and iSCSI logged out.
3. Build B target healthy.
4. Physical console and stable power.
5. BIOS settings recorded because updates can reset virtualization/IOMMU settings.

Also keep Proxmox kernels updated. Intel states that `e1000e` fixes are delivered through the upstream kernel rather than a separately maintained current driver package: [Intel Linux Ethernet driver information](https://www.intel.com/content/www/us/en/support/articles/000005480/ethernet-products.html).

Do not remove the workaround immediately after updating. Observe stability under iSCSI and Frigate load, then test one feature at a time only during a maintenance window.

## If it recurs

Use local console:

```bash
dmesg -T | grep -Ei 'e1000e|Hardware Unit Hang|NETDEV WATCHDOG' | tail -100
ethtool -k nic0
ethtool --show-eee nic0
```

Reapply the runtime recovery commands. If recovery fails, cleanly shut down VM1 before rebooting Proxmox:

```bash
qm shutdown 102 --timeout 90
qm status 102
reboot
```

Avoid a forced power cycle while VM1 has a writable iSCSI filesystem.

