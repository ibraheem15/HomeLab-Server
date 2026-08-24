# 11 — Troubleshooting Index

## Network and installation

| Symptom | Most likely cause | Resolution |
|---|---|---|
| USB not listed on multiple machines | Bad/incomplete image write | Verify ISO checksum and rewrite USB in raw image mode |
| Debian graphical installer goes black | Display-mode compatibility | Reboot and select plain text **Install** |
| “Bad archive mirror” in VM installer | VLAN tag `1` on untagged LAN | Clear Proxmox NIC VLAN field |
| DNS resolves but HTTPS cannot connect | Missing IPv4 default route | Add/correct `gateway 192.168.1.254` |
| Route works until reboot only | Added with `ip route`, not OS config | Fix `/etc/network/interfaces` |
| Both Proxmox and VM1 vanish | Build A physical network/power fault | Use local console; inspect `dmesg` and NIC |
| `e1000e ... Hardware Unit Hang` | Intel NIC transmit hardware/driver stall | Disable TSO/GSO/GRO + EEE, cycle link, persist post-up commands |

## Proxmox and VM lifecycle

| Symptom | Cause | Resolution |
|---|---|---|
| VM accidentally became template | Proxmox conversion action | Create a full clone with a new VMID; verify; later delete template |
| Template “uses RAM” | Misunderstanding | Template consumes disk only while stopped; no running CPU/RAM |
| Need another VM disk | Free storage required, not full-disk format | Allocate from existing free thin-pool/VG or intentionally add another pool |
| qemu guest agent/SSH shows AF_VSOCK warning | Optional systemd SSH vsock generator | Ignore if normal TCP SSH and guest agent work |
| TrueNAS installer console is unreadable/corrupted | Virtual framebuffer compatibility | Use SPICE display for installation; storage is unaffected |
| TrueNAS pool creation says duplicate serial `None` | Both emulated SCSI disks lack unique serials | Stop VM; add distinct `serial=` values to `scsi0` and `scsi1`; restart |

## TrueNAS, SMB, and VM2

| Symptom | Cause | Resolution |
|---|---|---|
| TrueNAS shows only one NIC | `net1` was absent or added without a full power cycle | Add VirtIO `net1` on `vmbr1`; fully stop/start VM `101` |
| VM2 shows only `ens18` | Storage NIC was not attached | Add VirtIO `net1` on `vmbr1`; fully stop/start VM `103` |
| VM2 has both `.30` and DHCP `.116` | Per-interface `dhcpcd@ens18` still owns a lease | Disable/mask the template unit or reboot after persistent static configuration; verify no `proto dhcp` route |
| Static IP works but DNS fails | `dhcpcd` previously generated `/etc/resolv.conf` | Add persistent nameservers to `/etc/resolv.conf`; retain `dns-nameservers` in interfaces file |
| CIFS mount works manually but not after boot | No persistent `_netdev`/systemd automount entry | Add the documented `/etc/fstab` line; daemon-reload; reboot-test |
| CIFS write returns permission denied | TrueNAS ACL, SMB credential, or UID/GID mapping mismatch | Verify `vm2_business` owner/group, restricted ACL, root-only credential file, and client UID/GID 1000 |
| TrueNAS quota blocks writes unexpectedly | Dataset reached its current adjustable limit | Increase quota if pool capacity permits; do not add a reservation |
| SFTPGo SQLite says `unable to open database file` | Container UID 1000 cannot write local `state` directory | `chown -R 1000:1000 state`; mode 750; recreate container |
| Tunnel shows `Unknown private service` | Route created as private hostname or published route not yet propagated | Use Published application route to `http://sftpgo:8080`; wait for Cloudflare propagation |
| Tunnel returns 502 | Connector cannot resolve/reach origin | Confirm both containers share `sftpgo_default`; test `http://sftpgo:8080/web/client` from that network |

## iSCSI

| Symptom | Cause | Resolution |
|---|---|---|
| TCP 3260 unreachable | Target/portal/network down | Check Build B address, `targetcli ls`, and `ss -lntp` |
| Login rejected | IQN ACL or CHAP mismatch | Compare exact initiator IQN and node credentials |
| LUN appears as `/dev/sdc` instead of `/dev/sdb` | Dynamic SCSI enumeration | Use UUID/PARTUUID, never device letters |
| LUN write-protected | Backstore or mapped-LUN read-only flag | Inspect targetcli backstore and ACL mapping; reconnect after correction |
| `targetcli` appears frozen | Suspended targetcli jobs after `Ctrl+Z` | Kill shell jobs; exit targetcli with `exit`/`Ctrl+D` |
| Two mounts shown at same path | Previous unmount failed; new mount stacked above it | `cd ~`; unmount repeatedly until `findmnt` is empty; mount once |
| `umount: target is busy` | Shell/process working inside mount | `cd ~`; run `fuser -vm`; stop owning process |
| iSCSI fails after 120 seconds | Default replacement timeout expired | Persist `node.session.timeo.replacement_timeout = 600` |

## Filesystem and HDD

| Symptom | Meaning | Action |
|---|---|---|
| `cat: Input/output error` | Block read failed; not permissions | Stop application work; inspect iSCSI/ext4 kernel logs |
| ext4 “aborted journal” | Writes failed while filesystem mounted | Cleanly disconnect VM1 and run offline recovery procedure |
| `findmnt` says `rw`, writes fail | Displayed mount flags do not reflect aborted journal usability | Trust kernel errors; do not remount over the problem |
| `e2fsck: ... is in use`, exit 8 | LIO backstore still holds device | Save target config; `targetctl clear`; then fsck |
| e2fsck extent tree “could be shorter” under `-n` | Optional optimization | No corruption action required |
| SMART pending/reallocated/uncorrectable > 0 | Physical media deterioration likely | Refresh backup, extended test, plan disk replacement |

## Frigate

| Symptom | Cause | Resolution |
|---|---|---|
| Compose fails on `/dev/dri/renderD128` | iGPU not passed into VM1 | Remove device mapping/hwaccel or complete passthrough first |
| Frigate starts but uses high CPU | CPU detector + software decode | Expected current mode; measure before considering GPU passthrough |
| UI unavailable at old port 5500 | Old mapping removed | Use authenticated `https://192.168.1.20:8971` |
| Config validation rejects `record.retain` | Old retention schema | Use `motion`, `alerts.retain`, and `detections.retain` |
| Recordings land on VM1 root | iSCSI source missing or wrong bind | Stop container immediately; verify mount and Docker mount source |
| Container refuses to start when storage is absent | `create_host_path: false` safety behavior | Restore iSCSI mount; do not bypass by creating a local directory |
| Existing events/history absent | `frigate.db` not copied with config | Stop container; restore complete offline config directory backup |

## Cloudflare Tunnel, Caddy, and private HTTPS

| Symptom | Cause | Resolution |
|---|---|---|
| Cloudflare route cannot reach `localhost:80` | `localhost` refers to `cloudflared`, not Caddy | Put both containers on `edge`; route to `http://caddy:80` |
| `400 The plain HTTP request was sent to HTTPS port` | Caddy uses HTTP to Frigate while Frigate TLS remains enabled | Set top-level `tls.enabled: false` in active Frigate config; restart Frigate |
| Two tunnels named `camera` and `services-vm` | Service route was mistakenly created as another tunnel | Keep active `services-vm`; move/verify route; delete inactive tunnel |
| `camera.example.com` follows private wildcard | Exact tunnel DNS record missing | Restore exact proxied CNAME to active tunnel; exact records override wildcard |
| Private hostname resolves to Cloudflare/tunnel | Obsolete exact record shadows wildcard | Remove exact route/CNAME; wildcard must be DNS-only A to VM1 Tailscale IP |
| `ERR_SSL_PROTOCOL_ERROR` | TLS listener/certificate failed before backend routing | Verify DNS, Caddy state, Cloudflare module, `127.0.0.1:8443`, and `tailscale serve status` |
| Caddy says Cloudflare module unregistered | Stock Caddy image still running | Build from Dockerfile and force-recreate Caddy |
| DNS-01 authentication/permission error | Missing/incorrect token or scope | Pass `CF_API_TOKEN`; require zone DNS Edit + zone Read for the specific zone |
| Caddy exited with status `0` after a power loss | The host sent external `SIGTERM`; backend refusal/DNS errors occurred while other containers were stopping or starting | Treat this as graceful host shutdown, then verify Caddy, cloudflared, Tailscale Serve, and each backend separately |
| `cannot assign requested address` on Caddy startup | Historical Compose mapping bound Docker directly to a Tailscale `100.x` IP before it existed | Use loopback `127.0.0.1:8443` plus persistent Tailscale Serve raw TCP `:443`; do not restore the direct bind |
| Private hostnames time out while Caddy is running | Tailscale is offline or its persistent Serve route is absent | Check `tailscale serve status`; expected route is tailnet TCP `:443` to `tcp://127.0.0.1:8443` |
| Trusted HTTPS followed by `502` | TLS works; Caddy cannot reach application | Attach web container to `edge`; verify container name, port, and backend scheme |
| `edge` declared but Caddy cannot resolve service | Service did not explicitly join `edge` | Add `networks: [default, edge]` under the service and recreate it |

## Immich

| Symptom | Cause | Resolution |
|---|---|---|
| PostgreSQL says data directory exists but is not empty | New bind directory contains files or failed initialization debris | Stop stack; quarantine directory; recreate an empty `postgres` directory |
| Entire Immich deployment directory becomes UID `999` | `DB_DATA_LOCATION` points to `immich/` instead of `immich/postgres/` | Stop via absolute Compose paths; quarantine directory; rebuild layout; correct `.env` |
| Immich reports `ENOTFOUND database` while PostgreSQL is restarting | Database container has no stable Docker DNS entry | Fix the PostgreSQL failure first |
| Immich reports `ENOTFOUND database` while PostgreSQL is healthy | Frontend and database do not share the app-private network | Attach both to `immich-internal`; recreate containers |
| Onboarding says `profile` has missing files | Database references an absent account avatar | Restore exact file from backup or continue and upload a new avatar; `.immich` marker is not a substitute |
| Valkey warns `Memory overcommit must be enabled` | Host kernel default is unsuitable for background save | Persist `vm.overcommit_memory=1` in `/etc/sysctl.d/99-valkey.conf` |

## AdGuard Home and Tailscale DNS

| Symptom | Cause | Resolution |
|---|---|---|
| `adguard.example.com` returns HTTP 502 | Caddy cannot reach the host-network UI | Proxy to `host.docker.internal:8083`; retain Caddy's `host.docker.internal:host-gateway` mapping |
| Host port 8083 does not answer after the wizard | Wizard saved `http.address` as `0.0.0.0:80`, while Docker mapped 8083→8083 | Stop AdGuard Home; set `http.address: 0.0.0.0:8083` in `AdGuardHome.yaml`; validate; start again |
| AdGuard Home cannot bind port 53 | Pi-hole, Technitium, or another resolver still owns the port | Stop the old resolver, set its restart policy to `no`, and prove port 53 is free before recreating AdGuard Home |
| All queries appear from one `172.x` client | Docker bridge source NAT hides original clients | Use `network_mode: host`; remove staging `ports`; recreate and verify distinct LAN/Tailscale addresses |
| A domain resolves even though blocking is enabled | The requested name and CNAME targets are absent from enabled lists, a list failed to update, or a public secondary handled the query | Check filtering/query log and list-update status before treating it as a resolver failure |
| Initial query takes about 45 ms | Cold cache/upstream round trip | Repeat the same query; warm-cache responses should be much faster before upstream changes are considered |
| Internet works but blocking is inconsistent | Public secondary DNS may be selected while AdGuard Home is healthy | Expected availability tradeoff; remove public secondary only if strict filtering is more important than DNS failover |
| VM1 resolves through itself or DNS loops | DNS host accepts the tailnet DNS policy | Run `sudo tailscale set --accept-dns=false` on VM1 |
| DNS uses AdGuard Home without an exit node | Tailnet DNS policy is independent of default-route selection | Expected: only DNS traverses VM1; web/download traffic remains direct |
| Selecting exit node does not change public IP | Exit-node advertisement or admin approval missing | Enable IP forwarding, advertise the exit node, approve it in Tailscale admin, then select `services-vm` on the client |

## Netdata and direct service access

| Symptom | Cause | Resolution |
|---|---|---|
| Compose says `network_mode` and `networks` are mutually exclusive | Netdata declares host mode and `edge` together | Keep `network_mode: host`; remove `networks`; proxy Caddy to `host.docker.internal:19999` |
| Proxmox shows VM memory near 100%, but applications are responsive | Linux filesystem cache is counted as used | Check `free -h`; use `available`, swap, and OOM logs rather than the free column |
| `service.domain.com` works but `TAILSCALE_IP:port` does not | Caddy reaches the container through `edge`, but no host port is published | Add a Compose `ports` mapping or keep domain-only access intentionally |
| Direct port gives TLS/protocol error | Raw service port is HTTP while Caddy supplies HTTPS for the domain | Use `http://TAILSCALE_IP:port`, unless the application itself provides HTTPS |

## Backrest and removable USB repository

| Symptom | Cause | Resolution |
|---|---|---|
| Backrest offers to initialize a new repository | Container sees an empty `/repo` or wrong path | Stop; mount USB by UUID; verify `config` + `is_mounted`; recreate Backrest |
| USB disk changes from `/dev/sdb1` to `/dev/sdc1` | Dynamic enumeration | Use UUID `ea3f8b0d-<SECRET>-e4029de67a34`, never device letters |
| USB appears on Build A but not VM1 | Proxmox USB passthrough does not match device/port | Verify VM102 USB mapping, enclosure identity, physical port, cable, and power |
| `e2fsck -n` reports free block/inode count mismatch | Read-only check skipped journal recovery | Unmount fully; run offline `e2fsck -f -p`; inspect exit code |
| `backup-usb stop` refuses to unmount | Restic child process is active | Wait for plan/check completion; do not force-remove the disk |
| Unmount says target busy | Shell or process holds the repository | Run `sudo fuser -vm /mnt/backup-seagate`; leave/stop the holder and retry |
| New snapshot seems likely to duplicate old backups | Snapshot paths/plans changed | Restic deduplicates unchanged content chunks within the existing repository |

### Reverse-proxy/TLS diagnostic bundle

```bash
cd /home/services-vm/services/proxy
sudo docker compose ps
sudo docker compose logs --tail=150 caddy
sudo docker compose logs --tail=100 cloudflared
sudo docker compose exec caddy \
  caddy list-modules | grep dns.providers.cloudflare
sudo docker inspect caddy \
  --format '{{json .HostConfig.PortBindings}}'
sudo ss -lntp | grep ':8443'
sudo tailscale serve status
sudo docker network inspect edge
```

From a Tailscale-connected client:

```bash
dig +short portainer.example.com
openssl s_client \
  -connect portainer.example.com:443 \
  -servername portainer.example.com </dev/null
curl -vkI https://portainer.example.com
```

Expected private DNS result: VM1's stable `100.x` address. Remove token values before sharing logs.

## High-signal diagnostic bundles

### Proxmox host

```bash
ip -br link
ip -4 -br address
ip route
bridge link
systemctl --failed
journalctl -b -u networking.service --no-pager -n 100
dmesg -T | grep -Ei 'e1000e|link|watchdog|reset|hang' | tail -100
```

### VM1 storage

```bash
findmnt /mnt/storage-b
sudo iscsiadm -m session -P 1
lsblk -o NAME,MAJ:MIN,SIZE,RO,FSTYPE,MOUNTPOINTS,MODEL
sudo journalctl -k -b --no-pager |
grep -Ei 'iscsi|timed out|I/O error|Buffer I/O|EXT4-fs|aborted journal' |
tail -150
```

### Build B target/HDD

```bash
sudo targetcli sessions
sudo targetcli ls
findmnt /dev/sdb2
sudo smartctl -H /dev/sdb
sudo smartctl -A /dev/sdb
sudo journalctl -k -b --no-pager |
grep -Ei 'I/O error|ata|reset|failed|sd[a-z]' |
tail -100
```

Remove secrets before sharing diagnostic output publicly.
