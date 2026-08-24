# 10 — Operations and Recovery Runbook

## Normal health check

### Build A / Proxmox

```bash
ip -4 -br address
ip route
ping -c 3 192.168.1.254
qm status 102
lvs
systemctl --failed
dmesg -T | grep -Ei 'e1000e|Hardware Unit Hang|NETDEV WATCHDOG' | tail -20
```

### Build B / target

```bash
hostnamectl
ip -4 -br address
findmnt /dev/sdb2
sudo targetcli sessions
sudo targetcli ls
sudo smartctl -H /dev/sdb
```

`findmnt /dev/sdb2` must be empty while VM1 uses the LUN.

### VM1

```bash
ip -4 -br address
ip route
sudo iscsiadm -m session -P 1
findmnt /mnt/storage-b
sudo test -w /mnt/storage-b && echo 'storage writable'
docker ps
cd /home/services-vm/frigate && docker compose ps
```

DNS health on VM1:

```bash
cd /home/services-vm/services/adguardhome
sudo docker compose ps
sudo ss -lntup | grep -E ':(53|8083)\b'
dig @127.0.0.1 example.com A
dig +tcp @192.168.1.20 example.com A
```

Expected: AdGuard Home is running in host mode, owns TCP/UDP 53, answers its UI on 8083, and resolves through both UDP and TCP. Pi-hole and Technitium must remain stopped with restart policy `no`. Do not start either while AdGuard Home owns port 53.

Reverse-proxy health on VM1:

```bash
cd /home/services-vm/services/proxy
sudo docker compose ps
sudo ss -lntp | grep ':8443'
sudo tailscale serve status
```

Expected: Caddy and cloudflared are running, Caddy is published only on `127.0.0.1:8443`, and Tailscale Serve forwards tailnet TCP `:443` to `tcp://127.0.0.1:8443`.

### TrueNAS and VM2

```bash
# Proxmox
qm status 101
qm status 103

# VM2
ip -4 -br address
ip route
ping -c 3 10.20.0.10
findmnt /mnt/vm2-business
df -hT /mnt/vm2-business
cd /home/baadesaba/services/sftpgo
sudo docker compose ps
sudo docker compose logs --tail=30 cloudflared
```

## Safe startup order

1. Router/switch available.
2. Build B boots; target restore service loads LIO configuration.
3. Confirm Build B listens on `192.168.1.51:3260`.
4. Build A boots.
5. TrueNAS VM `101` starts and imports pool `sata`.
6. VM1 `102` boots; open-iscsi logs in automatically and systemd mounts the UUID.
7. Docker starts Caddy independently on `127.0.0.1:8443`; cloudflared reconnects when public internet returns.
8. Tailscale restores its persistent raw TCP Serve route on tailnet `:443` when Tailscale becomes available.
9. VM2 `103` boots; its CIFS automount connects on first access.
10. VM3 `104` starts only after its configuration is completed.

AdGuard Home starts through Docker's `unless-stopped` policy after VM1 networking is available. Its tailnet DNS path has not yet been proven by a full VM1 reboot; perform and record that test only during an approved maintenance window.

After an uncontrolled outage, do not assume the mount is healthy merely because it exists. Run the VM1 health check and scan kernel logs.

The proxy path deliberately has no direct Docker bind to the Tailscale `100.x` address. If private HTTPS does not recover, check `tailscale serve status`; do not add a per-stack startup script or restore the old direct bind as a shortcut. A full reboot test of this corrected path remains a maintenance-window task.

## Safe shutdown order

Stop applications first:

```bash
cd /home/services-vm/frigate
docker compose down
```

Then VM1:

```bash
cd ~
sudo umount /mnt/storage-b
sudo iscsiadm -m node \
  -T <TARGET_IQN> \
  -p <STORAGE_SERVER_LAN_IP>:3260 --logout
sudo poweroff
```

Only after VM1 is off/no session may Build B be shut down or its HDD be checked locally.

For a normal Proxmox shutdown, use the UI or:

```bash
qm shutdown 102 --timeout 90
shutdown -h now
```

## After any network interruption

Inside VM1:

```bash
sudo iscsiadm -m session -P 1
findmnt /mnt/storage-b
sudo journalctl -k -b --no-pager |
grep -Ei 'iscsi|I/O error|Buffer I/O|EXT4-fs error|aborted journal' |
tail -100
```

If there are no fatal I/O/ext4 messages, test a temporary write. If there is an aborted journal or any `Input/output error`, stop services and follow [08-incident-iscsi-ext4-recovery.md](08-incident-iscsi-ext4-recovery.md).

## Backup cycle

1. Stop or quiesce databases/applications as required for consistency.
2. Power on the external backup HDD.
3. Confirm the expected repository and destination device by UUID/model.
4. Run Backrest/restic backup.
5. Run `restic check` according to repository size and maintenance policy.
6. Periodically perform test restores, not only repository scans.
7. Confirm schedule remains enabled.
8. Unmount/eject and power off the external HDD.

Minimum backup scope:

```text
VM1 local:
  /home/services-vm/frigate/config
  /home/services-vm/services/adguardhome/conf
  /home/services-vm/services/adguardhome/work
  future Vaultwarden data
  future Immich PostgreSQL dumps/data strategy
  Compose files and secret-management files

Build B media:
  irreplaceable photographs and application media
  Frigate footage only according to desired retention/importance
```

Container images and Debian packages are reproducible and lower priority than configuration, databases, and personal data.

## SMART maintenance

Monthly quick health:

```bash
sudo smartctl -H /dev/sdb
sudo smartctl -A /dev/sdb
```

Schedule SMART self-tests only when no heavy recording workload is expected:

```bash
sudo smartctl -t short /dev/sdb
sudo smartctl -l selftest /dev/sdb
```

Escalate immediately if any of these rise above zero:

- `Reallocated_Sector_Ct`
- `Current_Pending_Sector`
- `Offline_Uncorrectable`
- `UDMA_CRC_Error_Count` (often cable/path rather than platter)

The existing HDD has high start/stop and load-cycle counts. Avoid unnecessary power cycling of Build B itself; only the separate backup HDD is intentionally normally off.

## Capacity monitoring

### Proxmox

```bash
pvesm status
lvs
vgs
df -hT /
```

### TrueNAS/VM2

Check the TrueNAS dashboard for pool `sata` status, dataset usage, alerts, and SMART state. Inside VM2:

```bash
df -hT /mnt/vm2-business
sudo docker system df
du -sh /home/baadesaba/services/sftpgo/state
```

Do not fill ZFS beyond roughly 80–90%. Quotas can exceed physical capacity because they are limits, not reservations; monitor actual pool use.

### VM1 and iSCSI media

```bash
df -hT /
df -hT /mnt/storage-b
docker system df
```

Do not allow the VM1 root disk or Proxmox thin pool to approach exhaustion. Frigate’s media cleanup protects its recording tree but does not manage unrelated files in the old HDD root.

## Configuration backups before editing

```bash
sudo cp -a /etc/network/interfaces /etc/network/interfaces.before-change
sudo cp -a /etc/fstab /etc/fstab.before-change
sudo targetcli saveconfig
sudo cp -a /etc/rtslib-fb-target/saveconfig.json /root/saveconfig.before-change.json
```

Use descriptive, dated names during real maintenance. Never publish the target JSON because it contains CHAP credentials.

## Upgrade policy

- Apply Debian/Proxmox security updates regularly.
- Read Proxmox release notes before kernel upgrades.
- Update Docker Engine and Compose from the configured official apt repository.
- Pin or review major application upgrades; back up databases/config first.
- Pulling `frigate:stable` can change the application version. Read Frigate release/migration notes before recreating the container.
- Keep Tailscale current.
- Update Lenovo BIOS in a dedicated maintenance window, not while troubleshooting another subsystem.
