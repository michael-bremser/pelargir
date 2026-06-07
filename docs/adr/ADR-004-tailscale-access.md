# ADR-004: Tailscale via Barazinbar for Remote Access

**Status:** Accepted  
**Date:** 2026-06-07  
**Author:** Mike

---

## Context

The Jellyfin library needs to be accessible when traveling — from a phone or laptop on any network. This requires a mechanism to reach a service on the home LAN (10.28.99.0/24) from an external network without opening ports on Khazad-dûm (pfSense).

The finai stack uses Cloudflare Tunnel for its remote access layer. That approach is explicitly ruled out here: Cloudflare's terms of service prohibit using tunnels to proxy video or large media content, and all traffic through Cloudflare Tunnel is relayed via Cloudflare's network — adding latency and a third-party dependency that is unacceptable for video streaming.

---

## Decision

**Run Tailscale as a subnet router on a dedicated VM (Barazinbar) on Nogrod. Advertise the 10.28.99.0/24 subnet on the tailnet. Access Jellyfin at its LAN IP from anywhere on the tailnet.**

---

## Options Considered

### Option A — Cloudflare Tunnel

**Rejected.** Cloudflare's ToS prohibits streaming video through their tunnels. All traffic is proxied via Cloudflare's infrastructure — not peer-to-peer. Unacceptable for this workload on both legal and performance grounds.

### Option B — Open port on Khazad-dûm (port forwarding)

Forward port 8096 on the router to the jellyfin VM. Access via public IP or dynamic DNS.

**Pros:**
- No VPN software required on client devices
- Simple to configure

**Cons:**
- Exposes a port on the public internet — requires keeping Jellyfin patched, monitored, and behind a reverse proxy with TLS
- No access control at the network layer — anyone who finds the port can attempt to authenticate
- Not a pattern that maps to production infrastructure or Kubernetes — it is the thing production environments exist to avoid
- Dynamic DNS adds another dependency

### Option C — Tailscale on Khazad-dûm (pfSense package)

Install the Tailscale package on pfSense. Configure it as a subnet router advertising 10.28.99.0/24.

**Pros:**
- One less VM — Khazad-dûm already runs 24/7
- No additional provisioning

**Cons:**
- pfSense CE 2.8's NAT implementation uses Endpoint-Dependent Mapping ("hard" NAT), which forces Tailscale connections through DERP relay servers instead of establishing direct peer-to-peer links. Relayed video streaming has significantly higher latency and is unreliable for media playback
- The pfSense Tailscale package has known bugs in CE 2.7/2.8 where subnet routes silently drop from the kernel routing table, requiring manual service restarts to recover
- pfSense CE 2.9 is expected to ship with EIM-NAT support which would resolve the direct connection issue — see Future Reconsideration below
- FreeBSD package constraints mean the Tailscale version on pfSense lags behind the Linux client

### Option D — Tailscale on dedicated Barazinbar VM (chosen)

Provision a minimal Ubuntu VM on Nogrod (Barazinbar). Run Tailscale in subnet router mode, advertising 10.28.99.0/24. All tailnet devices reach the entire LAN subnet through this single router.

**Pros:**
- Native Linux Tailscale client — full feature support, direct peer-to-peer connections, no DERP relay requirement
- No NAT issues — the VM has a standard Linux network stack with Tailscale's recommended configuration
- One subnet router covers the entire 10.28.99.0/24 network — adding new services (Jellyfin, or anything else on the VLAN) requires zero additional Tailscale configuration
- Dedicated VM means one job, one process, easily snapshotted and documented
- Traffic is peer-to-peer between the client device and Barazinbar — Tailscale's mesh VPN model means no third-party infrastructure handles the video stream

**Cons:**
- One additional VM to maintain (minimal — 1 vCPU, 512MB RAM, 8GB disk)
- Nogrod carries two additional VMs (jellyfin + Barazinbar) alongside k3s-control — headroom check required

---

## Reasoning

The video streaming requirement eliminates any solution that proxies traffic through a third party. Cloudflare Tunnel is out. The pfSense Tailscale package's current NAT behavior makes DERP relay fallback likely, which is unacceptable for streaming. Port forwarding exposes attack surface unnecessarily.

Barazinbar is a tiny VM — it runs a single process (tailscaled) and consumes negligible resources. The cost is low. The benefit is a stable, direct-connection subnet router with a full Linux Tailscale client and no known reliability issues.

The single subnet router pattern is deliberate. Rather than running Tailscale on every VM or container that needs remote access, Barazinbar advertises the entire subnet. Any device on 10.28.99.0/24 is reachable from the tailnet automatically. This maps cleanly to how production ingress works — one entry point, not per-service configuration.

---

## k8s Migration Path

In the Kubernetes phase, the subnet router pattern is replaced by the Tailscale Kubernetes operator. The operator integrates with the cluster and exposes services on the tailnet directly, replacing both Barazinbar and the need for direct port access to the jellyfin VM.

| Docker Phase | Kubernetes Phase |
|---|---|
| Barazinbar VM advertises 10.28.99.0/24 subnet | Tailscale Kubernetes operator installed in cluster |
| Access Jellyfin at `10.28.99.5x:8096` via tailnet | Access Jellyfin via tailnet service name via operator |
| Barazinbar running 24/7 | Tailscale operator runs as a pod — no separate VM |
| Subnet router pattern | Service-level tailnet exposure |

Barazinbar is retired when the cluster takes over. The tailnet itself (devices, ACLs, auth keys) does not change — only the mechanism that bridges the cluster to it.

---

## Future Reconsideration — pfSense CE 2.9

pfSense CE 2.9 is expected to ship with EIM-NAT (Endpoint-Independent Mapping NAT) support, which would allow Tailscale on Khazad-dûm to establish direct peer-to-peer connections without DERP relay fallback. When 2.9 is released:

- Verify EIM-NAT is implemented and stable in the release notes
- Test direct connection confirmation (`tailscale status` should show direct, not relay)
- Verify subnet route stability under normal operation (no route drops observed)
- If both criteria are met: migrate subnet router role to Khazad-dûm and retire Barazinbar

This decision should be revisited within 90 days of a stable pfSense CE 2.9 release.

---

## Consequences

- Barazinbar VM provisioned on Nogrod: 1 vCPU, 512MB RAM, 8GB disk, Ubuntu 24.04 LTS, VLAN 99
- Tailscale installed on Barazinbar in subnet router mode
- Route `10.28.99.0/24` advertised and approved in Tailscale admin console
- IP forwarding enabled on Barazinbar (`net.ipv4.ip_forward = 1`)
- Tailscale auth key stored securely — not committed to this repository
- No ports opened on Khazad-dûm for Jellyfin
- All tailnet devices reach Jellyfin at its LAN IP with no additional per-device configuration
