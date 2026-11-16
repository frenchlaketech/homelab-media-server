
# Homelab

A self-hosted media stack running on a single small-form-factor PC with Proxmox. Everything runs in LXC containers, downloads are routed through a VPN, and media is stored on a mirrored ZFS pool.

## Overview

| Layer | Details |
|-------|---------|
| Hypervisor | Proxmox VE |
| Workloads | LXC containers (Docker host + Jellyfin) |
| Storage | ZFS mirror (`MediaPool`), ~16TB raw across two 8TB drives |
| Networking | VPN-gated download client (Gluetun + PIA), AdGuard for DNS (planned) |
| Media server | Jellyfin with Intel iGPU hardware transcoding |

## Hardware

- **Host:** HP EliteDesk 400 (SFF, KBL/Q270 board)
- **CPU:** Intel Core i5-6500
- **RAM:** 24GB
- **Boot/OS:** 1TB NVMe
- **Media storage:** 2x 8TB drives attached directly to the host (not a separate NAS)
- **GPU:** GTX 1070 considered, but case fit was uncertain; transcoding uses the Intel iGPU instead

## Storage Layout

`MediaPool` is a ZFS mirror mounted at `/mnt/media` on the Proxmox host.

```
/mnt/media
├── Movies    # existing library
├── Shows     # existing library
└── data      # unified dataset bind-mounted into containers as /data
    ├── torrents/
    └── media/
```

- A single `data` dataset holds both downloads and the final library, so Radarr/Sonarr can **hardlink** instead of copying (verified with a hardlink test).
- Files are owned by UID/GID `1000`; containers map to this UID so permissions line up across the stack.

## Containers

| CT | Name | Resources | Purpose |
|----|------|-----------|---------|
| 100 | `ArrStack` | 4 cores, 8GB RAM, 32GB disk | Docker host for the arr apps, download client and VPN |
| 101 | Jellyfin | LXC | Media server, read-only `/data/media` mount, iGPU passthrough |
| *** | Tailscale | LXC | VPN and remote access to server |

### ArrStack (CT 100)

- Docker with a `media` user (UID 1000)
- TUN device passthrough and UID 1000 mapping configured in the LXC config
- `/mnt/media/data` bind-mounted to `/data`
- Compose file at `/opt/arr/docker-compose.yaml` with secrets in a `.env` file

**Services**

- **Gluetun**: VPN client (OpenVPN, Private Internet Access, US Chicago)
- **qBittorrent**: download client, runs behind Gluetun so all torrent traffic goes through the VPN
- **Prowlarr**: indexer management
- **Radarr / Sonarr**: movie and TV automation
- **Seerr**: request front-end for movies and TV (port 5055)

### Jellyfin

- Intel iGPU passthrough (render group GID 104 inside the container)
- Hardware transcoding working
- Library mounted read-only

## Workflow

```
Request (Seerr)
     │
     ▼
Radarr / Sonarr ──── Prowlarr (indexers)
     │
     ▼
qBittorrent (behind Gluetun / PIA VPN)
     │   downloads to /data/torrents
     ▼
Hardlink into /data/media (Movies / Shows)
     │
     ▼
Jellyfin (read-only, hardware transcoding) ──► playback
```

1. **Request:** a movie or show is requested in Seerr.
2. **Search:** Radarr/Sonarr query indexers via Prowlarr.
3. **Download:** the release is sent to qBittorrent, which only has network access through the Gluetun VPN tunnel.
4. **Import:** on completion, the file is hardlinked into the media library on the same dataset (no duplicate space used).
5. **Serve:** Jellyfin picks up the new item and streams it, transcoding on the iGPU when needed.

## Design Decisions

- **Single Docker host for all the arrs** rather than one LXC per app, to keep management simple.
- **VPN container as the network gateway** for the download client, so nothing leaks if the tunnel drops.
- **ZFS mirror** for redundancy on the media pool.
- **One shared `/data` dataset** to enable hardlinks and atomic moves.
- **Release sizes capped** in Radarr and Sonarr to avoid huge (~20GB) releases; ~8GB 1080p is the target for movies.

## Status / TODO

- [x] ZFS mirror healthy (`MediaPool`)
- [x] ArrStack LXC with Docker, Gluetun and qBittorrent
- [x] Jellyfin with hardware transcoding
- [x] Seerr request system
- [x] Existing library copied into `/data`
- [ ] Fix wrong names/covers for existing library items in Jellyfin
- [x] Finish Prowlarr indexers (some blocked by Cloudflare)
- [x] Release size limits set for movies and series
- [ ] AdGuard DNS
- [ ] Tailscale container for remote access
- [ ] Email when updates are Available?
- [ ] Reverse proxy for remote viewing of Media
