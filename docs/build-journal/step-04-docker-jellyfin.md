# Build Journal — Step 4: Docker and Jellyfin

**Date:** 6/13/2026
**Host:** pelargir VM (10.28.99.50)  
**Status:** [x] Completed

---

## Objective

Install Docker on pelargir, deploy Jellyfin via compose with the iGPU and NFS mounts wired in, and verify hardware transcoding end to end. At the end of this step the media server works on the LAN.

**Prerequisites:** Step 2 (`vainfo` clean, render/video GIDs recorded) and Step 3 (both NFS mounts surviving reboot).

---

## Why Compose, Why These Choices

Everything in the compose file is written to map to a k8s manifest later:

| Compose construct | k8s equivalent |
|---|---|
| `services.jellyfin` | Deployment |
| bind mounts to NFS paths | PVCs on `aglarond-nfs` |
| `devices: /dev/dri` | `resources.limits: gpu.intel.com/i915: "1"` + Intel device plugin |
| `group_add` GIDs | pod `securityContext.supplementalGroups` |
| `restart: unless-stopped` | `restartPolicy: Always` |
| `network_mode: host` | Service + nginx Ingress (replaced, not translated) |

No Docker-managed named volumes anywhere — data lives in NFS paths the VM merely mounts. The container, like the VM, holds nothing.

---

## Steps

### 1. Install Docker (official repo, not the snap, not the Ubuntu package)

```bash
# Remove anything stale
sudo apt remove -y docker.io docker-compose 2>/dev/null

# Docker's official repo
sudo apt update && sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Run docker without sudo
sudo usermod -aG docker $USER
```

Log out and back in (or `newgrp docker`), then verify:

```bash
docker run --rm hello-world
```

### 2. Create the compose directory

```bash
mkdir -p ~/jellyfin && cd ~/jellyfin
```

The compose file is tracked in the repo at `compose/jellyfin/docker-compose.yml` — edit it there and copy it here, or symlink. The repo is the source of truth, not the VM.

### 3. Write the compose file

```bash
vim docker-compose.yml
```

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped

    # Host networking: simplest path for client discovery (DLNA/SSDP)
    # and avoids a NAT hop on streaming traffic. Replaced by
    # Service + Ingress in the k8s phase, not translated.
    network_mode: host

    # Quick Sync: hand the render node to the container
    devices:
      - /dev/dri:/dev/dri

    # GIDs verified in Step 2 §16 on this VM: render=993, video=44.
    # If the VM is ever rebuilt, re-run `getent group render` /
    # `getent group video` and update — these are NOT stable across installs.
    group_add:
      - "993"   # render
      - "44"    # video

    volumes:
      # Config: Jellyfin DB, metadata, settings → NFS (survives VM rebuild)
      - /mnt/jellyfin-config:/config
      # Media library → NFS, read-only; Jellyfin never writes to media
      - /mnt/media:/media:ro
      # Transcode scratch: LOCAL disk, deliberately not NFS — see note
      - /var/cache/jellyfin-transcodes:/cache

    environment:
      - TZ=America/Los_Angeles
```

> **Why the transcode cache is local, not NFS:** transcoding writes temporary segment files at high frequency. Putting that churn on NFS adds latency to the hot path and hammers Aglarond for data that is garbage within minutes. Scratch space belongs on local disk; it is the one category of data that *should* die with the VM. Create it: `sudo mkdir -p /var/cache/jellyfin-transcodes`

> **Why media is `:ro`:** Jellyfin has no reason to write to the library. Gundabad's rip workflow is the only writer. Read-only is free insurance against a misclick in the UI deleting media.

### 4. Start it

```bash
docker compose up -d
docker compose logs -f --tail=50   # Ctrl+C to detach; watch for clean startup
```

### 5. Initial Jellyfin setup

Browse to `http://10.28.99.50:8096` from Gundabad:

1. Create the admin account
2. Add library → Movies → folder `/media/movies`
3. Add library → Shows → folder `/media/tv`
4. Skip remote access settings (Tailscale handles this at the network layer — Step 5)

### 6. Enable Quick Sync transcoding

```
Dashboard → Playback → Transcoding
```

- **Hardware acceleration:** Intel QuickSync (QSV)
- **QSV Device:** `/dev/dri/renderD128`
- Enable hardware decoding for: H264, HEVC, VP9 (leave AV1 *decode* on if listed; UHD 630 has no AV1 *encode*)
- **Enable hardware encoding:** checked
- Leave tone-mapping off for now (UHD 630 can do it via VPP but verify basic transcode first)

Save.

### 7. Verify hardware transcoding end to end

Two terminals:

**Terminal 1 — watch the GPU on pelargir:**

```bash
sudo intel_gpu_top
```

**Terminal 2 / browser — force a transcode:**

Play any file and manually lower the quality (playback settings → e.g. 480p) so Jellyfin must transcode rather than direct-play.

Success criteria:
- `intel_gpu_top` shows the **Video** engine busy (this is the Quick Sync fixed-function block)
- VM CPU stays low (`htop` — single-digit percentages, not pegged cores)
- Playback starts within a few seconds and seeks cleanly

Also confirm in the Jellyfin dashboard: the active stream should report "Transcoding (hw)".

If the Video engine sits at 0% while CPU spikes, Jellyfin fell back to software — check the container has `/dev/dri` (`docker exec jellyfin ls -l /dev/dri`) and that the GIDs in `group_add` match this VM's actual render/video groups.

### 8. Commit the compose file to the repo

The tested compose file goes into `compose/jellyfin/docker-compose.yml` in the pelargir repo (sanitize nothing — it contains no secrets). This file is the k8s migration source document.

---

## Verification

- [x] `docker run hello-world` clean, docker runs without sudo
- [x] `docker compose up -d` starts Jellyfin with no errors in logs
- [x] Web UI reachable at `http://10.28.99.50:8096` from the LAN
- [x] Movies + TV libraries created against `/media/movies` and `/media/tv`
- [x] QSV enabled, device `/dev/dri/renderD128`
- [x] Forced transcode plays; `intel_gpu_top` Video engine active; CPU low
- [x] Dashboard shows "Transcoding (hw)" on the test stream
- [x] Compose file committed to repo
- [x] Container survives `docker compose down && docker compose up -d` with config intact (proves config truly lives on NFS)

---

## Next Step

[Step 5 — Tailscale Subnet Router on Barazinbar](step-05-tailscale-barazinbar.md)
