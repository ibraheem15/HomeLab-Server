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

## SMB storage mount

The TrueNAS share is `//10.20.0.10/vm3-extra`, accessed with the TrueNAS account `vm3_extra`.

### Install the client and create the mount point

```bash
sudo apt update
sudo apt install -y cifs-utils
sudo mkdir -p /mnt/vm3-extra
```

Confirm the numeric UID and GID of the VM3 Linux user who should own files on the mounted share:

```bash
id
```

The examples below assume the intended user has UID `1000` and GID `1000`. Replace those numbers if `id` reports different values.

### Store the SMB username and password

Create a root-only credentials file:

```bash
sudo install -m 600 /dev/null /root/.smb-vm3-extra
sudoedit /root/.smb-vm3-extra
```

Enter the TrueNAS SMB credentials—not the VM3 Linux login password:

```text
username=vm3_extra
password=<TRUENAS_SMB_PASSWORD>
```

Confirm that only root can read the file:

```bash
sudo chown root:root /root/.smb-vm3-extra
sudo chmod 600 /root/.smb-vm3-extra
sudo stat -c '%U:%G %a %n' /root/.smb-vm3-extra
```

The expected mode is `root:root 600`.

### Test a manual mount

First confirm that VM3 can reach TrueNAS over the storage network, then mount the share:

```bash
ping -c 3 10.20.0.10
sudo mount -t cifs \
  //10.20.0.10/vm3-extra \
  /mnt/vm3-extra \
  -o credentials=/root/.smb-vm3-extra,vers=3.1.1,uid=1000,gid=1000,file_mode=0660,dir_mode=0770
```

Verify the mount and test write access as the intended non-root user:

```bash
findmnt /mnt/vm3-extra
touch /mnt/vm3-extra/vm3-write-test
ls -l /mnt/vm3-extra/vm3-write-test
rm /mnt/vm3-extra/vm3-write-test
```

Unmount the manual test before configuring persistence:

```bash
sudo umount /mnt/vm3-extra
```

### Configure persistent automount

Back up and edit `/etc/fstab`:

```bash
sudo cp -a /etc/fstab /etc/fstab.pre-vm3-smb
sudoedit /etc/fstab
```

Add this single line, replacing `uid=1000,gid=1000` if the intended user's numeric IDs differ:

```fstab
//10.20.0.10/vm3-extra /mnt/vm3-extra cifs credentials=/root/.smb-vm3-extra,vers=3.1.1,uid=1000,gid=1000,file_mode=0660,dir_mode=0770,_netdev,nofail,x-systemd.automount,x-systemd.after=network-online.target,x-systemd.mount-timeout=30s 0 0
```

`nofail` allows VM3 to boot while TrueNAS is unavailable. `x-systemd.automount` mounts the share on first access. Reload systemd, validate the entry, and trigger the automount:

```bash
sudo systemctl daemon-reload
sudo mount -a
ls /mnt/vm3-extra
findmnt /mnt/vm3-extra
```

Finally, reboot VM3 and verify that the share still mounts on access:

```bash
sudo reboot
```

After reconnecting:

```bash
ls /mnt/vm3-extra
findmnt /mnt/vm3-extra
```

## Remaining work

1. Confirm both guest NIC names and configure static addresses.
2. Remove any residual DHCP lease and verify DNS persistence.
3. Follow the SMB storage procedure above; manually test the credential and write access.
4. Verify the persistent `/etc/fstab` automount after a reboot.
5. Install QEMU guest agent, SSH, updates, and Docker only as required.
6. Join VM3 to extra's separate Tailscale organization—not the existing owner tailnet.
7. Decide whether the hardware owner retains only Proxmox console access or an additional VM administrator path.
8. Enable Start at boot only after storage and shutdown behavior are verified.
