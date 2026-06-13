# Build Journal — Step 3: NFS Mounts from Aglarond

**Date:** 6/12/26
**Hosts:** Aglarond (TrueNAS UI) for Part 1; pelargir VM for Part 2  
**Status:** [x] Completed

---

## Objective

Create the datasets and NFS exports on Aglarond, mount them in the pelargir VM, and establish the media folder structure. After this step the VM has persistent storage that survives rebuilds — which Step 1's events proved is not a theoretical concern.

**Prerequisite:** Step 2 complete — `vainfo` verified. Do storage before Docker so the compose file in Step 4 can reference mounts that already exist and are already tested.

---

## Why This Layout

Two separate concerns, two separate mounts:

- **Media** (`/mnt/media`) — large, read-mostly, shared. Movies and TV. Gundabad's ripping workflow writes here directly over NFS; Jellyfin reads it.
- **Config** (`/mnt/jellyfin-config`) — small, write-heavy, private to Jellyfin. Its database, metadata cache, and settings.

Keeping them separate means the ZFS snapshot policy can differ (config needs frequent snapshots, media barely changes), and in the k8s phase they map to two distinct PVCs instead of one tangled volume. See ADR-003.

---

## Part 1 — Aglarond (TrueNAS UI)

### 1. Create the datasets

```
Datasets → MainPool → Add Dataset
```

Create two:

| Dataset | Purpose | Record size |
|---------|---------|-------------|
| `MainPool/media` | Movie/TV library | 1M (large sequential files) |
| `MainPool/jellyfin-config` | Jellyfin DB + metadata | 128K (default) |

> Record size on the media dataset: 1M suits large video files read sequentially. Not critical, but free performance.

### 2. Create the NFS shares

```
Shares → UNIX (NFS) Shares → Add
```

For each dataset:

- **Path:** `/mnt/MainPool/media` (and `/mnt/MainPool/jellyfin-config`)
- **Networks:** `10.28.99.0/24` and `10.28.20.10` — restrict to VLAN 99 and Gundabad
- **Maproot User / Group:** `root` / `root` for the config share. For the media share, map to the user that owns the media files so Gundabad's writes and pelargir's reads agree on ownership.

Enable the NFS service if it isn't already running (`System Settings → Services → NFS`).

### 3. Note the export paths

TrueNAS SCALE exports as `/mnt/MainPool/<dataset>`. Verify from any Linux machine:

```bash
showmount -e 10.28.11.10
```

Both paths must appear before continuing.

---

## Part 2 — pelargir VM

### 4. Install NFS client tools

```bash
sudo apt install -y nfs-common
```

### 5. Create mount points

```bash
sudo mkdir -p /mnt/media /mnt/jellyfin-config
```

### 6. Test-mount by hand first

Never go straight to fstab — test interactively so failures are visible:

```bash
sudo mount -t nfs 10.28.11.10:/mnt/MainPool/media /mnt/media
sudo mount -t nfs 10.28.11.10:/mnt/MainPool/jellyfin-config /mnt/jellyfin-config
df -h | grep mnt
```

Write test:

```bash
sudo touch /mnt/media/.write-test && sudo rm /mnt/media/.write-test
sudo touch /mnt/jellyfin-config/.write-test && sudo rm /mnt/jellyfin-config/.write-test
```

If a write fails with permission denied, fix the maproot settings on the Aglarond share — do not chmod your way around it from the client.

### 7. Make the mounts persistent

```bash
sudo vim /etc/fstab
```

Append:

```
10.28.11.10:/mnt/MainPool/media            /mnt/media            nfs  defaults,_netdev,nofail  0  0
10.28.11.10:/mnt/MainPool/jellyfin-config  /mnt/jellyfin-config  nfs  defaults,_netdev,nofail  0  0
```

**Why these options:**
- `_netdev` — wait for the network before mounting; without it boot can race the NIC and fail
- `nofail` — **the VM must boot even if Aglarond is down.** Without `nofail`, an unreachable NFS server leaves the VM hanging in early boot — and after Step 1's adventures, no mount gets the power to brick a boot

Verify fstab parses and mounts cleanly:

```bash
sudo umount /mnt/media /mnt/jellyfin-config
sudo systemctl daemon-reload
sudo mount -a
df -h | grep mnt
```

### 8. Reboot test

```bash
sudo reboot
```

After boot, confirm both mounts came up on their own:

```bash
df -h | grep mnt
```

### 9. Create the media folder structure

Jellyfin's metadata matching depends on this layout — establish it before the first file lands:

```bash
sudo mkdir -p /mnt/media/movies /mnt/media/tv
```

Naming convention for everything that goes in (this is the contract with the ripping workflow on Gundabad):

```
/mnt/media/movies/Film Title (Year)/Film Title (Year).mkv
/mnt/media/tv/Show Name/Season 01/Show Name - S01E01.mkv
```

> Gundabad mounts the same media export for the rip → encode → drop workflow. Files written there in this structure are picked up by Jellyfin automatically once the library is configured in Step 4. Ripping can begin as soon as this step is done — it does not wait for Jellyfin.

---

## k8s Migration Note

These two fstab mounts are the Docker-phase stand-in for two PVCs on the `aglarond-nfs` StorageClass already running in the finai cluster. The NFS server, exports, and data do not move at migration time — only the mounting layer changes (fstab → PV/PVC). This is the entire reason the VM holds no data.

---

## Verification

- [x] Both datasets exist on Aglarond with NFS shares restricted to 10.28.99.0/24
- [x] `showmount -e 10.28.11.10` lists both exports
- [x] Both mounts succeed by hand, reads and writes verified
- [x] fstab entries include `_netdev,nofail`
- [x] Mounts survive a reboot without intervention
- [x] `movies/` and `tv/` structure created
- [x] Gundabad can mount the media export and write to it (ripping workflow unblocked)

---

## Next Step

[Step 4 — Docker and Jellyfin](step-04-docker-jellyfin.md)
