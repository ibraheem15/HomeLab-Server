# 25 — VM2 Network and Storage

VM 103: Debian 13, 4 vCPU, 4 GiB RAM, 20 GiB local disk; LAN 192.168.1.30/24 with gateway 192.168.1.254; storage NIC 10.20.0.20/24 with no gateway. Confirm both Proxmox VirtIO NICs are present before guest configuration.

Install CIFS client and test the share manually before persisting it:

    sudo apt install cifs-utils
    sudo mkdir -p /mnt/vm2-business
    sudo mount -t cifs //10.20.0.10/vm2-business /mnt/vm2-business -o credentials=/root/.smb-vm2,uid=<VM2_UID>,gid=<VM2_GID>,vers=3.1.1
    findmnt /mnt/vm2-business

Store the credentials file root-readable only. Use a systemd-aware fstab automount after manual test, then verify it survives a reboot. SFTPGo state, SQLite, keys, and Compose live locally; only uploads use the SMB mount. Configure Docker and Tailscale after both networks and mount work.
