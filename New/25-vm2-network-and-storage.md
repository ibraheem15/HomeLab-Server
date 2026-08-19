# 25 — VM2 Network and Storage

## VM definition

| Setting | Value |
|---|---|
| VMID | `103` |
| Hostname | `baadesaba` |
| OS | Debian 13 minimal |
| CPU | 4 host-type vCPU |
| RAM | 4096 MiB; balloon minimum 2048 MiB |
| Local disk | 20 GiB SCSI on `local-lvm` |
| `net0` | VirtIO `ens18` on `vmbr0` |
| `net1` | VirtIO `ens19` on `vmbr1` |

Twenty GiB is sufficient because the VM stores Debian, Docker images, SFTPGo state, and logs locally; uploaded files live on TrueNAS. Docker log rotation is mandatory.

## Missing second NIC

Debian initially showed only `ens18`. Proxmox VM `103` lacked or had not power-cycled after adding `net1`:

```bash
qm shutdown 103
qm set 103 -net1 virtio,bridge=vmbr1,firewall=0
qm config 103 | grep '^net'
qm start 103
```

After a full start, `ens19` appeared.

## Persistent network configuration

`/etc/network/interfaces`:

```text
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto ens18
iface ens18 inet static
        address 192.168.1.30/24
        gateway 192.168.1.254
        dns-nameservers 192.168.1.254 1.1.1.1

auto ens19
iface ens19 inet static
        address 10.20.0.20/24
```

Expected routes:

```text
default via 192.168.1.254 dev ens18
192.168.1.0/24 dev ens18
10.20.0.0/24 dev ens19
```

### Duplicate DHCP address

After applying static networking, `ens18` briefly held both `.30` and DHCP `.116`, with a `proto dhcp` default route. A per-interface `dhcpcd` process still managed the lease. A reboot after persistent configuration cleared the duplicate in this installation. If it returns:

```bash
systemctl status dhcpcd@ens18.service --no-pager
sudo systemctl disable --now dhcpcd@ens18.service
sudo systemctl mask dhcpcd@ens18.service
```

Verify that only `.30` remains and no route says `proto dhcp`.

### DNS loss

Stopping DHCP removed the resolver configuration that `dhcpcd` had generated. `/etc/resolv.conf` was restored with:

```text
nameserver 192.168.1.254
nameserver 1.1.1.1
nameserver 9.9.9.9
```

Test both routing and DNS separately:

```bash
ping -c 3 1.1.1.1
getent hosts debian.org
```

## Manual SMB validation

Packages and paths:

```bash
sudo apt install -y cifs-utils
sudo mkdir -p /mnt/vm2-business
sudo install -m 600 /dev/null /root/.smb-vm2
```

`/root/.smb-vm2`:

```text
username=vm2_business
password=<SECRET>
```

Manual mount:

```bash
sudo mount -t cifs \
  //10.20.0.10/vm2-business \
  /mnt/vm2-business \
  -o credentials=/root/.smb-vm2,vers=3.1.1,uid=1000,gid=1000,file_mode=0660,dir_mode=0770
```

Read/write/delete tests passed without `sudo`.

## Persistent automount

`/etc/fstab` entry:

```fstab
//10.20.0.10/vm2-business /mnt/vm2-business cifs credentials=/root/.smb-vm2,vers=3.1.1,uid=1000,gid=1000,file_mode=0660,dir_mode=0770,_netdev,nofail,x-systemd.automount,x-systemd.after=network-online.target,x-systemd.mount-timeout=30s 0 0
```

`nofail` allows boot if TrueNAS is unavailable. `x-systemd.automount` connects on first access. After every storage/network change:

```bash
sudo systemctl daemon-reload
sudo mount -a
findmnt /mnt/vm2-business
```

## Guest baseline

Installed/enabled:

```text
qemu-guest-agent
openssh-server
Tailscale (administration only; no exit-node/subnet advertisement)
Docker Engine + Compose plugin from Docker's official Debian repository
```

VM2 start order is after TrueNAS. Docker `/etc/docker/daemon.json` limits local logs:

```json
{
  "log-driver": "local",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```
