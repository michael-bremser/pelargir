# ADR-003: NFS on Aglarond for Media and Config Persistence

**Status:** Accepted  
**Date:** 2026-06-07  
**Author:** Mike

---

## Context

Jellyfin has two categories of persistent data: the media library (video files, music, photos — read-heavy, large) and application config (database, metadata cache, transcoding settings, user accounts — small, read/write). Both need to survive VM rebuilds.

The jellyfin VM is intentionally disposable — it can be snapshotted, rolled back, or rebuilt from scratch. Any storage solution that ties data to the VM violates this design principle.

Aglarond (TrueNAS) is already running in the homelab, serving NFS to the finai k3s cluster. It has an existing ZFS pool with available capacity.

---

## Decision

**Mount NFS shares from Aglarond for both media and config. The VM itself holds no persistent data.**

---

## Options Considered

### Option A — Local VM disk

Store media and config on the VM's virtual disk inside Proxmox.

**Pros:**
- Zero additional configuration — data is just in the VM
- Fast local I/O

**Cons:**
- Data is coupled to the VM. Rebuild or migrate the VM and the library is gone
- Proxmox VM disk is not a backup target — TrueNAS ZFS is
- Violates the compute/storage separation principle that makes the finai stack resilient
- No path to Kubernetes — k8s cannot access data stored inside a VM disk

### Option B — NFS from Aglarond (chosen)

Mount two NFS shares from Aglarond into the jellyfin VM. Bind-mount them into the Docker container.

**Pros:**
- Data is completely independent of the VM — rebuild the VM, data is untouched
- Aglarond's ZFS pool provides checksumming, compression, and snapshot scheduling — real data integrity without cluster-level configuration
- The same Aglarond NFS export already serves the finai k3s cluster. No new infrastructure required
- The k8s migration path is direct: NFS host-level mounts become PersistentVolumeClaims backed by the `aglarond-nfs` StorageClass already deployed in the finai cluster. The NFS paths do not change — only the mechanism for mounting them changes
- Media library is accessible from any future VM or cluster node without copying data

**Cons:**
- NFS is a network dependency — if Aglarond is unreachable, Jellyfin cannot serve media. Mitigated by ZFS redundancy on Aglarond
- Slightly higher latency than local disk for config read/write — not perceptible for Jellyfin's workload

---

## Reasoning

The fundamental requirement is that the VM is disposable. This is non-negotiable for a homelab where VMs get rebuilt, Proxmox gets upgraded, and experiments happen. Any media or config stored inside the VM is at risk every time the VM is touched.

Aglarond already exists and already serves NFS. The ZFS dataset structure is already established from the finai stack. This is not a new dependency — it is using existing infrastructure for its designed purpose.

The k8s migration path is important. In the Docker phase, the NFS paths are mounted at the OS level and bind-mounted into the container. In Kubernetes, those same NFS paths become PVC definitions pointing at the `aglarond-nfs` StorageClass. The data does not move. The NFS server does not change. The only thing that changes is the layer that manages the mount. This is exactly the right abstraction boundary to understand before working with k8s persistent storage in production.

---

## NFS Share Layout

| Share | NFS Path | Mount Point in VM | Purpose |
|-------|----------|-------------------|---------|
| Media library | `aglarond:/mnt/MainPool/media` | `/mnt/media` | Video, music, photos |
| Jellyfin config | `aglarond:/mnt/MainPool/jellyfin/config` | `/mnt/jellyfin/config` | DB, metadata, settings |

---

## Docker → k8s Migration Mapping

| Docker Phase | Kubernetes Phase |
|---|---|
| NFS mounted in VM via `/etc/fstab` | PersistentVolume pointing at same NFS path |
| Bind-mounted into container via `volumes:` in compose | PersistentVolumeClaim using `aglarond-nfs` StorageClass |
| `ReclaimPolicy` not applicable (host mount) | `ReclaimPolicy: Retain` on PVC — data not deleted on PVC removal |

The NFS paths on Aglarond do not change between phases. The media library and config are at the same location before and after the migration.

---

## Consequences

- NFS client tools (`nfs-common`) installed on jellyfin VM
- Two NFS mounts configured via `/etc/fstab` on the VM with `_netdev` flag (wait for network before mounting)
- Docker compose file uses bind mounts to the VM's NFS mount points — no Docker-managed volumes
- Aglarond ZFS snapshot schedule covers `MainPool/media` and `MainPool/jellyfin`
- VM rebuild procedure: provision new VM, install NFS client, mount shares, run Docker compose — library and config restored automatically
