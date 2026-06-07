# ADR-001: VM over LXC for Jellyfin

**Status:** Accepted  
**Date:** 2026-06-07  
**Author:** Mike

---

## Context

Jellyfin needs to run somewhere on the Proxmox cluster. Proxmox offers two runtimes: full KVM virtual machines and LXC containers. The workload has two constraints that make this decision non-trivial: Intel Quick Sync hardware transcoding requires direct access to the iGPU, and the deployment is explicitly designed to migrate to Kubernetes later.

---

## Decision

**Run Jellyfin in Docker inside a KVM VM. Do not use LXC.**

---

## Options Considered

### Option A — LXC Container

Proxmox LXC containers share the host kernel. They are lighter than VMs and start faster.

**Pros:**
- Lower overhead than a full VM — no hardware emulation layer
- Faster provisioning
- `/dev/dri` device passthrough to LXC is documented

**Cons:**
- iGPU passthrough to LXC requires manually binding `/dev/dri/renderD128` into the container config — fragile, not well-documented for Quick Sync specifically, and breaks across Proxmox version updates
- LXC has no migration path to Kubernetes. A container running Jellyfin directly in LXC produces no transferable knowledge about Docker volumes, compose files, or the patterns that translate to k8s manifests
- No snapshot support for running containers — snapshots require stopping the LXC first, and even then behavior is less reliable than VM snapshots
- Not how any production workload runs — learning the wrong mental model early

### Option B — KVM VM with Docker (chosen)

Provision a standard Ubuntu VM. Install Docker. Run Jellyfin as a Docker container.

**Pros:**
- iGPU passthrough via VFIO is the standard, documented path for KVM — well-understood, stable across Proxmox versions
- Every Docker concept learned (volumes, compose files, networking, bind mounts) maps directly to Kubernetes primitives — volumes → PVCs, compose services → Deployments, bind mounts → ConfigMaps or NFS PVs
- Full VM snapshots — snapshot before any risky operation, instant rollback
- Clean separation: Proxmox manages hardware and VM lifecycle, Docker manages the workload. Each layer does one job
- Portable — the Proxmox host is invisible to Docker. Migrate the VM to different hardware, or export and reimport it, without touching any container configuration

**Cons:**
- Higher overhead than LXC — a full Ubuntu install consumes more RAM and disk than an LXC container
- Slightly more provisioning work upfront

---

## Reasoning

The iGPU passthrough requirement settles the performance argument. KVM VFIO passthrough is mature, documented, and stable. LXC device passthrough for Quick Sync has persistent edge cases and requires manual Proxmox config file editing that can break silently on upgrades.

The migration path argument settles everything else. This deployment is explicitly designed to become a Kubernetes workload. Every hour spent learning Docker volumes, compose networking, and container restart policies is an hour that directly shortens the k8s migration. LXC produces none of that — it is a dead end from a learning perspective.

The reasoning is identical to ADR-003 in the finai stack: isolation, snapshot capability, and a hypervisor layer that is transparent to the workload.

---

## Consequences

- jellyfin VM provisioned on Nogrod — KVM, Ubuntu 24.04 LTS, iGPU passthrough
- Docker installed on the VM — Jellyfin runs as a Docker container
- All Docker configuration (compose file, volume mounts, environment variables) is written to be translatable to k8s manifests without a full rewrite
- VM can be snapshotted before any significant change
- Migration to Kubernetes (Phase 3) is a manifest-writing exercise — not a rebuild
