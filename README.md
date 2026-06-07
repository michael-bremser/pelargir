# 📺 Pelargir — Self-Hosted Media Server

> A fully local media server running Jellyfin on a Proxmox VM with Intel Quick Sync hardware transcoding. Media lives on Aglarond. Access from anywhere via Tailscale. Designed from the start to migrate to Kubernetes without a rebuild.

---

## What This Is

A production-pattern self-hosted media server built as a deliberate DevOps learning project. Jellyfin runs in Docker on a Proxmox VM on Nogrod. Hardware transcoding via Intel Quick Sync (UHD 630 iGPU passthrough). Persistent data on Aglarond (TrueNAS NFS). Remote access via Tailscale subnet router on Barazinbar.

Every architectural decision is documented. The Docker phase is not the end state — it is a deliberate stepping stone to a Kubernetes deployment, and every decision is made with that migration in mind.

---

## Why It Exists

Two goals:

1. **Daily utility** — a private media library that actually gets used, with hardware transcoding, available at home and on the road
2. **Career development** — every component is a deliberate teaching moment. Docker networking, volume mounts, iGPU passthrough, NFS persistence, and Tailscale routing all map directly to concepts that appear in production infrastructure and Kubernetes deployments

---

## Naming Convention

Infrastructure nodes follow a Middle-earth Dwarf place name theme. This repo is named after Pelargir, the great Gondorian port city — a distribution point for everything passing through.

---

## Stack

| Component | Role | Runtime |
|-----------|------|---------|
| [Jellyfin](https://jellyfin.org) | Media server — streaming, transcoding, library management | Docker |
| Intel UHD 630 | Hardware transcoding via Quick Sync | iGPU passthrough to VM |
| Aglarond (TrueNAS) | Media library and config persistence | NFS mount |
| Barazinbar | Tailscale subnet router — remote access | Dedicated VM |
| Tailscale | Peer-to-peer VPN — travel access to media library | Tailscale daemon |

---

## Infrastructure

| Name | Thing | Hardware | Role |
|------|-------|----------|------|
| **Nogrod** | Proxmox node | Lenovo M70q · i3-10100T (UHD 630) · 32GB DDR4 | Hypervisor — hosts jellyfin VM and Barazinbar |
| **jellyfin** | VM on Nogrod | 4 vCPU · 8GB RAM · 32GB disk · Ubuntu 24.04 | Jellyfin + Docker · iGPU passthrough |
| **Barazinbar** | VM on Nogrod | 1 vCPU · 512MB RAM · 8GB disk · Ubuntu 24.04 | Tailscale subnet router |
| **Aglarond** | TrueNAS | ZFS · NFS export | Media library + config persistence |
| **Khazad-dûm** | pfSense | — | Router, firewall, internal DNS |

---

## Topology

```
Nogrod (10.28.99.11)
┌─────────────────────────────────────────────────┐
│  Proxmox Host                                   │
│                                                 │
│  ┌──────────────────────┐  ┌─────────────────┐  │
│  │  jellyfin VM         │  │  Barazinbar VM  │  │
│  │  Ubuntu 24.04        │  │  Ubuntu 24.04   │  │
│  │  10.28.99.5x         │  │  10.28.99.5x    │  │
│  │                      │  │                 │  │
│  │  Docker              │  │  tailscaled     │  │
│  │  └─ Jellyfin         │  │  subnet router  │  │
│  │                      │  │  → 10.28.99.0/24│  │
│  │  /dev/dri (iGPU)     │  └─────────────────┘  │
│  │  Intel UHD 630       │                       │
│  │  Quick Sync          │                       │
│  └──────────────────────┘                       │
└─────────────────────────────────────────────────┘
          │                        │
          │ NFS                    │ Tailscale
          ▼                        ▼
   Aglarond (TrueNAS)       Your devices
   /media      → library    anywhere on
   /config     → state      the tailnet
```

**Key design principle:** the VM is disposable. Media and config live on Aglarond over NFS. Rebuild the VM from scratch and Jellyfin comes back with the full library and all settings intact — the same compute/storage separation principle used in the finai stack.

---

## Access Patterns

| Location | How | Notes |
|----------|-----|-------|
| Home LAN | Direct — `http://10.28.99.5x:8096` | No Tailscale needed |
| Remote (travel) | Tailscale → LAN IP | Barazinbar advertises 10.28.99.0/24 — reach Jellyfin at its LAN IP |
| Jellyfin mobile app | Tailscale + LAN IP | Works identically to home once connected |

Tailscale traffic is peer-to-peer between your device and Barazinbar — no relay, no third-party proxy, no ToS constraint on video streaming. This is why Cloudflare Tunnel is explicitly not used here (see ADR-004).

---

## Build Phases

### Phase 0 — Infrastructure 🔜 Next
- Provision jellyfin VM on Nogrod (iGPU passthrough, VLAN 99)
- Provision Barazinbar VM on Nogrod (Tailscale subnet router)
- NFS mounts from Aglarond for media and config

### Phase 1 — Core Stack
- Docker installed on jellyfin VM
- Jellyfin container deployed with NFS volume mounts
- Intel Quick Sync hardware transcoding verified
- Tailscale running on Barazinbar, subnet route approved
- Internal DNS via Khazad-dûm (`jellyfin.local`)

### Phase 2 — Hardening
- Jellyfin behind nginx reverse proxy (local only)
- TLS via self-signed cert or local CA
- Watchtower for automated container updates
- Healthcheck and restart policies

### Phase 3 — Kubernetes Migration
- Jellyfin redeployed as a Kubernetes Deployment
- iGPU exposed via Intel GPU device plugin
- NFS volumes become PVCs backed by `aglarond-nfs` StorageClass
- Tailscale subnet router replaced by Tailscale Kubernetes operator
- nginx Ingress replaces direct port exposure
- Barazinbar VM retired

### Phase 4 — Future Consideration
- Reevaluate Khazad-dûm (pfSense) as Tailscale subnet router once pfSense CE 2.9 ships with EIM-NAT support — see ADR-004 for criteria

---

## Repository Structure

```
/
├── README.md
├── docs/
│   ├── adr/                              # Architecture Decision Records
│   │   ├── ADR-001-vm-over-lxc.md
│   │   ├── ADR-002-igpu-passthrough.md
│   │   ├── ADR-003-nfs-storage.md
│   │   └── ADR-004-tailscale-access.md
│   └── build-journal/                    # Step-by-step build notes
│       ├── step-01-proxmox-vms.md
│       ├── step-02-igpu-passthrough.md
│       ├── step-03-nfs-mounts.md
│       ├── step-04-docker-jellyfin.md
│       └── step-05-tailscale-barazinbar.md
├── compose/
│   └── jellyfin/
│       └── docker-compose.yml
└── scripts/
```

---

## Architecture Decisions

All major decisions are documented as Architecture Decision Records in `/docs/adr/`. Each ADR captures the context, the options considered, the decision made, and the reasoning. See [ADR-001](docs/adr/ADR-001-vm-over-lxc.md) to start.

---

## Build Journal

Step-by-step build notes live in `/docs/build-journal/`. Each entry covers what was done, what was learned, and what broke. Written during the build — not after.

---

## DevOps Concepts Covered

| Concept | Where |
|---------|-------|
| VM isolation from hypervisor | jellyfin and Barazinbar as Proxmox VMs |
| iGPU device passthrough | Intel UHD 630 → jellyfin VM via VFIO |
| Compute/storage decoupling | NFS mounts from Aglarond — VM is disposable |
| Docker volumes and bind mounts | NFS paths mounted into Jellyfin container |
| Reverse proxy pattern | nginx in front of Jellyfin (Phase 2) |
| Subnet routing | Barazinbar advertises LAN subnet on tailnet |
| VPN mesh networking | Tailscale peer-to-peer, no open ports |
| k8s migration path | Docker → Deployment, NFS mount → PVC, subnet router → Tailscale operator |

---

## Network

- **Router/Firewall:** Khazad-dûm (pfSense)
- **Internal DNS:** `jellyfin.local` via Khazad-dûm
- **VLAN:** All VMs on VLAN 99 (Valinor), 10.28.99.0/24
- **Remote access:** Tailscale — no open ports on Khazad-dûm

---

*Part of a broader homelab DevOps learning project. See also: [book-of-mazarbul](https://github.com/you/book-of-mazarbul) — private AI financial assistant stack.*
