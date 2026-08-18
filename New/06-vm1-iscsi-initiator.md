# 06 — VM1: iSCSI Initiator and Persistent ext4 Mount

## Install and identify the initiator

Inside VM1:

```bash
sudo apt update
sudo apt install open-iscsi
sudo cat /etc/iscsi/initiatorname.iscsi
```

Current initiator:

```text
InitiatorName=<INITIATOR_IQN>
```

Test the target port:

```bash
nc -vz <STORAGE_SERVER_LAN_IP> 3260
```

## Discover and configure CHAP

Discover:

```bash
sudo iscsiadm -m discovery -t sendtargets -p <STORAGE_SERVER_LAN_IP>
```

Configure the discovered node with the same CHAP username/password defined in Build B:

```bash
sudo iscsiadm -m node \
  -T <TARGET_IQN> \
  -p <STORAGE_SERVER_LAN_IP>:3260 \
  --op update -n node.session.auth.authmethod -v CHAP

sudo iscsiadm -m node \
  -T <TARGET_IQN> \
  -p <STORAGE_SERVER_LAN_IP>:3260 \
  --op update -n node.session.auth.username -v <CHAP_USERNAME>

sudo iscsiadm -m node \
  -T <TARGET_IQN> \
  -p <STORAGE_SERVER_LAN_IP>:3260 \
  --op update -n node.session.auth.password -v '<SECRET>'
```

Do not put the secret into shell history on a shared system. Restrict `/etc/iscsi` to root and store the credential in the password manager.

Enable automatic login and extend recovery time:

```bash
sudo iscsiadm -m node \
  -T <TARGET_IQN> \
  -p <STORAGE_SERVER_LAN_IP>:3260 \
  --op update -n node.startup -v automatic

sudo iscsiadm -m node \
  -T <TARGET_IQN> \
  -p <STORAGE_SERVER_LAN_IP>:3260 \
  --op update -n node.session.timeo.replacement_timeout -v 600
```

The original 120-second recovery timeout expired during the Intel NIC outage and returned fatal I/O errors to ext4. `600` makes storage I/O wait up to ten minutes for reconnection. Tradeoff: storage-dependent processes may block for ten minutes rather than fail immediately.

Login:

```bash
sudo iscsiadm -m node \
  -T <TARGET_IQN> \
  -p <STORAGE_SERVER_LAN_IP>:3260 --login
```

Verify:

```bash
sudo iscsiadm -m session -P 1
sudo lsblk -f
sudo blkid
```

## Device names are not stable

The LUN appeared first as `/dev/sdb`, then as `/dev/sdc` after logout/login while an older device object still existed. This is normal Linux enumeration behavior. Never use a `/dev/sdX` name in persistent configuration.

Authoritative filesystem UUID:

```text
<FILESYSTEM_UUID>
```

## First read-only inspection

```bash
sudo mkdir -p /mnt/storage-b
sudo mount -t ext4 -o ro,noload \
  UUID=<FILESYSTEM_UUID> \
  /mnt/storage-b
findmnt /mnt/storage-b
```

After validating data, unmount and configure normal read/write mounting.

## Persistent mount

`/etc/fstab` contains one line:

```fstab
UUID=<FILESYSTEM_UUID> /mnt/storage-b ext4 rw,_netdev,x-systemd.requires=open-iscsi.service,x-systemd.after=network-online.target,x-systemd.after=open-iscsi.service,x-systemd.device-timeout=90s,x-systemd.mount-timeout=90s 0 0
```

Apply and test:

```bash
sudo systemctl daemon-reload
sudo mount /mnt/storage-b
findmnt /mnt/storage-b
sudo touch /mnt/storage-b/.vm1-write-test
sudo rm /mnt/storage-b/.vm1-write-test
```

## Stacked-mount incident

An earlier read-only mount remained because `umount` was attempted while the shell’s working directory was inside `/mnt/storage-b`; its error was hidden with `2>/dev/null`. A new read/write mount was then placed on the same mountpoint, producing two `findmnt` rows:

```text
/mnt/storage-b /dev/sdb ro,norecovery
/mnt/storage-b /dev/sdc rw
```

Recovery:

```bash
cd ~
sudo umount /mnt/storage-b
findmnt /mnt/storage-b
sudo umount /mnt/storage-b
findmnt /mnt/storage-b
```

The first unmount removed the top layer; the second removed the hidden lower layer. After `findmnt` returned nothing, mount once from `fstab`.

Lesson: never suppress unmount errors during storage maintenance, and always leave the mounted directory before unmounting.

## Convenient home-directory access

Keep the canonical mount in `/mnt`; create a symlink instead of mounting over the home directory:

```bash
ln -s /mnt/storage-b /home/services-vm/storage-b
```

The symlink persists across reboot:

```text
/home/services-vm/storage-b -> /mnt/storage-b
```

Do not mount the LUN directly over `/home/services-vm`; doing so would hide local home-directory files and could disrupt login.

## Reboot verification

```bash
sudo reboot
```

After boot:

```bash
findmnt /mnt/storage-b
sudo iscsiadm -m session
sudo test -w /mnt/storage-b && echo 'Persistent iSCSI mount works'
sudo iscsiadm -m node -o show | grep -E 'node.startup|replacement_timeout'
```

Expected:

```text
node.startup = automatic
node.session.timeo.replacement_timeout = 600
```

