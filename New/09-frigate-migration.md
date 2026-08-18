# 09 — Frigate Migration to VM1

## Final storage boundary

```text
VM1 local SSD
/home/services-vm/frigate/compose.yaml
/home/services-vm/frigate/config/config.yml
/home/services-vm/frigate/config/frigate.db
/home/services-vm/frigate/config/<other config-side files>

Build B via iSCSI
/mnt/storage-b/home/<LEGACY_USER>/camera/frigate/storage/
  recordings/
  clips/snapshots
  exports
```

Frigate writes its SQLite database inside `/config`; therefore the entire config directory is local. Bulk media remains remote. Frigate documents `/config` as configuration + SQLite and `/media/frigate` as recording/snapshot storage: [Frigate installation and storage](https://docs.frigate.video/frigate/installation/).

## File migration

From the old tree:

```text
/mnt/storage-b/home/<LEGACY_USER>/camera/frigate/config/
```

copy the complete directory contents to:

```text
/home/services-vm/frigate/config/
```

Keep together if present:

- `config.yml` or `config.yaml`
- `frigate.db`
- `frigate.db-wal`
- `frigate.db-shm`
- model/cache/state files stored alongside the database

Rename `config.yaml` to `config.yml` and ensure only one exists; Frigate prefers `config.yml` if both are present.

Do not move or duplicate the old `storage/` directory. Bind it directly from its iSCSI path.

## Working `compose.yaml`

```yaml
services:
  frigate:
    container_name: frigate
    image: ghcr.io/blakeblackshear/frigate:stable
    restart: unless-stopped
    stop_grace_period: 30s
    shm_size: "512mb"

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
      - "8971:8971"
      - "8554:8554"
      - "8555:8555/tcp"
      - "8555:8555/udp"
```

### Why the old Compose was changed

| Old setting | Final setting | Reason |
|---|---|---|
| `privileged: true` | Removed | No hardware device currently requires broad privileges |
| `/dev/dri/renderD128` | Removed | iGPU is not passed into VM1 |
| `shm_size: 128mb` | `512mb` | More headroom for frame processing |
| `5500:5000` | `8971:8971` | Port 5000 is internal/unauthenticated; 8971 is authenticated UI |
| `1984:1984` | Removed | Avoid exposing go2rtc management/API without a requirement |
| Single `8555` mapping | TCP + UDP mappings | WebRTC uses both transports |
| Relative `./storage` | Absolute iSCSI bind | Makes the storage boundary explicit |
| Default bind creation | `create_host_path: false` | Fail safely if iSCSI media path is unavailable |
| Docker 2 CPU / 4 GB cap | Removed initially | VM-level 6-vCPU/24-GB boundary already protects host; CPU decode + OpenVINO need headroom |

Frigate’s official container image remains:

```text
ghcr.io/blakeblackshear/frigate:stable
```

## Working detector and model

The first stable deployment keeps object detection on CPU:

```yaml
model:
  path: /openvino-model/ssdlite_mobilenet_v2.xml
  width: 300
  height: 300
  labelmap_path: /openvino-model/coco_91cl_bkgr.txt

detectors:
  ov:
    type: openvino
    device: CPU
```

Do not enable this until `/dev/dri/renderD128` exists inside VM1:

```yaml
ffmpeg:
  hwaccel_args: preset-vaapi
```

For Intel generations 8–12, Frigate recommends `preset-vaapi`; there is no CPU fallback when explicitly configured hardware decoding fails. [Frigate video acceleration](https://docs.frigate.video/configuration/hardware_acceleration_video/).

## Camera structure retained

Two cameras are configured:

```text
camera_1  <CAMERA_1_LAN_IP>; ONVIF 8899
camera_2    <CAMERA_2_LAN_IP>; ONVIF 8899
```

Each camera uses a high-resolution stream for recording and a low-resolution stream for detection through go2rtc restreams at `127.0.0.1:8554`. Passwords must remain real secrets, not be committed to a public repository.

## Correct Frigate 0.16 retention schema

The old direct `record.retain` structure was replaced for each camera:

```yaml
record:
  enabled: true
  motion:
    days: 7
  alerts:
    retain:
      days: 7
      mode: motion
  detections:
    retain:
      days: 7
      mode: motion

snapshots:
  enabled: true
  retain:
    default: 7
```

Current Frigate separates continuous, motion, alert, and detection retention. [Recording configuration](https://docs.frigate.video/configuration/record/).

## Validate and start

```bash
cd /home/services-vm/frigate
docker compose config
docker compose pull
docker compose up -d
docker compose logs -f --tail=100
```

Open:

```text
https://<SERVICES_VM_LAN_IP>:8971
```

Validation checklist:

```bash
docker compose ps
docker inspect frigate --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
findmnt -T /mnt/storage-b/home/<LEGACY_USER>/camera/frigate/storage
df -hT /mnt/storage-b
```

Confirm `/config` resolves to VM1 local storage and `/media/frigate` resolves beneath `/mnt/storage-b`. Verify both camera live views, recordings, snapshots, object detections, retention cleanup, and container recovery after a controlled VM reboot.

## Current limitation

Frigate is working without GPU passthrough. This costs more VM CPU but preserves Proxmox’s physical display for emergency recovery. Measure CPU load before changing that tradeoff.

