# 27 — VM3 Installation Checkpoint

## Completed

VM3 has been created and Debian 13 installed.

| Setting | Value |
|---|---|
| VMID | `104` |
| Suggested VM name | `extra-vm` |
| Machine/firmware | Q35 + OVMF/UEFI |
| CPU | 4 cores, type `host` |
| RAM | 4096 MiB; balloon minimum 2048 MiB |
| Local disk | 30 GiB SCSI on `local-lvm` |
| `net0` | VirtIO on `vmbr0` |
| `net1` | VirtIO on `vmbr1` |
| Bulk dataset | `sata/vm3-extra`, initial quota 500 GiB |

VM3's local disk is for Debian, Docker images, configuration, and databases. Bulk files belong on TrueNAS.

## Target network configuration

```text
LAN interface:     192.168.1.40/24
Default gateway:   192.168.1.254
Storage interface: 10.20.0.30/24
Storage gateway:   none
TrueNAS storage:   10.20.0.10
```

Expected `/etc/network/interfaces` shape, after confirming actual interface names:

```text
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto ens18
iface ens18 inet static
        address 192.168.1.40/24
        gateway 192.168.1.254
        dns-nameservers 192.168.1.254 1.1.1.1

auto ens19
iface ens19 inet static
        address 10.20.0.30/24
```

Do not apply this blindly if interface names differ.

## Target storage mount

Future share:

```text
//10.20.0.10/vm3-extra
```

Use TrueNAS account `vm3_extra`, a root-only credentials file, and a persistent `_netdev,nofail,x-systemd.automount` CIFS mount. Map client ownership to the intended VM3 Linux UID/GID after confirming that user's numeric IDs.

## Remaining work

1. Confirm both guest NIC names and configure static addresses.
2. Remove any residual DHCP lease and verify DNS persistence.
3. Install `cifs-utils`; manually test the SMB credential and write access.
4. Add the persistent `/etc/fstab` automount and reboot-test it.
5. Install QEMU guest agent, SSH, updates, and Docker only as required.
6. Join VM3 to extra's separate Tailscale organization—not the existing owner tailnet.
7. Decide whether the hardware owner retains only Proxmox console access or an additional VM administrator path.
8. Enable Start at boot only after storage and shutdown behavior are verified.
