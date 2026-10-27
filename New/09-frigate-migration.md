# 09 — Frigate on VM1

## Storage boundary

| Path | Owner | Contents |
|---|---|---|
| `/home/services-vm/frigate/config` | VM1 local NVMe | config, SQLite, WAL/SHM, state |
| `/mnt/storage-b/home/<LEGACY_USER>/camera/frigate/storage` | Build B HDD via iSCSI | recordings, clips, snapshots, exports |

Copy the old config directory complete, including frigate.db, frigate.db-wal, frigate.db-shm, model/cache files, and exactly one config.yml. Do not copy or duplicate the storage directory; bind it from iSCSI.

## Compose pattern

```yaml
services:
  frigate:
    image: ghcr.io/blakeblackshear/frigate:stable
    container_name: frigate
    restart: unless-stopped
    stop_grace_period: 30s
    shm_size: 512mb
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /home/services-vm/frigate/config:/config
      - type: bind
        source: /mnt/storage-b/home/<LEGACY_USER>/camera/frigate/storage
        target: /media/frigate
        bind:
          create_host_path: false
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - 8971:8971
      - 8554:8554
      - 8555:8555/tcp
      - 8555:8555/udp
```

The fail-closed bind is intentional: an absent iSCSI mount must prevent startup instead of writing media into a new VM1 directory. Do not use privileged mode or a DRM device until GPU passthrough exists.

## Current detection and retention

```yaml
detectors:
  ov:
    type: openvino
    device: CPU
record:
  enabled: true
  motion:
    days: 7
  alerts:
    retain: { days: 7, mode: motion }
  detections:
    retain: { days: 7, mode: motion }
snapshots:
  enabled: true
  retain: { default: 7 }
```

CPU/OpenVINO is current. Do not add VAAPI until /dev/dri/renderD128 is passed through VM1; configured hardware decode has no CPU fallback. Frigate authenticated UI is 8971. Keep port 5000 and go2rtc administration unexposed. Attach Frigate to edge for Caddy access.

## Deploy and prove ownership

    findmnt -T /mnt/storage-b/home/<LEGACY_USER>/camera/frigate/storage
    cd /home/services-vm/frigate
    sudo docker compose config
    sudo docker compose pull
    sudo docker compose up -d
    sudo docker compose ps
    sudo docker inspect frigate --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'

Verify /config is local, /media/frigate is under /mnt/storage-b, both camera views/recordings work, and a controlled reboot does not create a local fallback. Use the current [Frigate configuration reference](https://docs.frigate.video/) only for version-specific schema changes.
