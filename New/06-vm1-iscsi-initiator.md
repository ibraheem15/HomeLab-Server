# 06 — VM1: iSCSI Initiator and Persistent ext4 Mount

VM1 is the only iSCSI initiator. Build B exports the HDD partition; VM1 mounts its filesystem at `/mnt/storage-b`. Never use an sdX name in persistent configuration.

## Discover, authenticate, and log in

    sudo apt update
    sudo apt install open-iscsi
    sudo cat /etc/iscsi/initiatorname.iscsi
    nc -vz 192.168.1.51 3260
    sudo iscsiadm -m discovery -t sendtargets -p 192.168.1.51

Configure the discovered target with its real target IQN and CHAP credentials:

    sudo iscsiadm -m node -T <TARGET_IQN> -p 192.168.1.51:3260 --op update -n node.session.auth.authmethod -v CHAP
    sudo iscsiadm -m node -T <TARGET_IQN> -p 192.168.1.51:3260 --op update -n node.session.auth.username -v <CHAP_USERNAME>
    sudo iscsiadm -m node -T <TARGET_IQN> -p 192.168.1.51:3260 --op update -n node.session.auth.password -v '<SECRET>'
    sudo iscsiadm -m node -T <TARGET_IQN> -p 192.168.1.51:3260 --op update -n node.startup -v automatic
    sudo iscsiadm -m node -T <TARGET_IQN> -p 192.168.1.51:3260 --op update -n node.session.timeo.replacement_timeout -v 600
    sudo iscsiadm -m node -T <TARGET_IQN> -p 192.168.1.51:3260 --login

The 600-second replacement timeout was chosen after the pve-a NIC incident: it gives a transient path failure time to recover, but storage processes can block for up to ten minutes.

Verify session and identify the filesystem UUID:

    sudo iscsiadm -m session -P 1
    sudo lsblk -f
    sudo blkid

## Inspect once, then mount persistently

For the first inspection only, mount read-only without journal replay:

    sudo mkdir -p /mnt/storage-b
    sudo mount -t ext4 -o ro,noload UUID=<FILESYSTEM_UUID> /mnt/storage-b
    findmnt /mnt/storage-b
    cd ~
    sudo umount /mnt/storage-b

Then add one fstab entry:

    UUID=<FILESYSTEM_UUID> /mnt/storage-b ext4 rw,_netdev,x-systemd.requires=open-iscsi.service,x-systemd.after=network-online.target,x-systemd.after=open-iscsi.service,x-systemd.device-timeout=90s,x-systemd.mount-timeout=90s 0 0

Apply and prove both the mount and write path:

    sudo systemctl daemon-reload
    sudo mount /mnt/storage-b
    findmnt /mnt/storage-b
    sudo touch /mnt/storage-b/.vm1-write-test
    sudo rm /mnt/storage-b/.vm1-write-test

Create only a symlink for the convenient home path:

    ln -s /mnt/storage-b /home/services-vm/storage-b

Do not mount over the home directory; that hides local files.

## Stacked mount recovery

If two mount rows appear, an earlier mount was not removed. Leave the directory, unmount until the lookup is empty, then mount only once:

    cd ~
    sudo umount /mnt/storage-b
    findmnt /mnt/storage-b
    sudo umount /mnt/storage-b
    findmnt /mnt/storage-b

Never hide unmount errors. Reboot verification requires a session, mount, writable test, and confirmation that startup is automatic and replacement timeout is 600. I/O errors or an aborted journal require [08](08-incident-iscsi-ext4-recovery.md), not a remount.
