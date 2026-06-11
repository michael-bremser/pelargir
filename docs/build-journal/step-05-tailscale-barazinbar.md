# Build Journal — Step 5: Tailscale Subnet Router on Barazinbar

**Date:** —  
**Host:** barazinbar VM (10.28.99.51)  
**Status:** [ ] Not Started

---

## Objective

Stand up Tailscale on Barazinbar as a subnet router advertising 10.28.99.0/24, so the Jellyfin library (and the whole VLAN) is reachable from any device on the tailnet while traveling. Phase 2 of the project — Jellyfin must already be working on the LAN (Step 4) before this adds anything.

See ADR-004 for why this is a dedicated VM and not the pfSense package (CE 2.8 NAT limitations force DERP relay fallback — unacceptable for video), and for the pfSense CE 2.9 reevaluation criteria.

---

## What a Subnet Router Actually Does

A normal Tailscale node exposes only itself to the tailnet. A subnet router additionally advertises routes to a LAN behind it — packets from your phone on the tailnet destined for 10.28.99.50 arrive at Barazinbar over WireGuard, get forwarded out its LAN interface, and the reply path reverses. Nothing else on the VLAN runs Tailscale; Barazinbar is the single bridge.

This is why IP forwarding was enabled in Step 1 — Barazinbar is a router, and Linux won't forward packets between interfaces without it.

---

## Steps

### 1. Pre-flight

```bash
# On barazinbar — confirm forwarding survived from Step 1
sysctl net.ipv4.ip_forward
# net.ipv4.ip_forward = 1
```

If it shows 0, redo Step 1 §8 before continuing — Tailscale will warn but the routes will silently not work.

### 2. Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

(Same `curl | sh` pattern as k3s — acceptable here for the same reason: official installer, inspectable, widely used. The apt repo it configures handles future updates.)

### 3. Bring it up as a subnet router

```bash
sudo tailscale up --advertise-routes=10.28.99.0/24 --hostname=barazinbar
```

- Authenticate via the printed URL using your tailnet account
- `--advertise-routes` offers the VLAN to the tailnet — it is not active until approved (next step)
- Deliberately **not** using `--advertise-exit-node`: this box routes traffic *to the homelab*, it does not become the default route for all device traffic. Narrower is better.

### 4. Approve the route in the admin console

```
https://login.tailscale.com/admin/machines
→ barazinbar → Edit route settings → approve 10.28.99.0/24
```

While there, two hygiene items:

- **Disable key expiry** for barazinbar (machine menu → Disable key expiry). Otherwise the node silently drops off the tailnet when the key expires in ~180 days — discovered, inevitably, while traveling.
- Confirm the node shows **"Subnet router"** tag in the machine list.

### 5. Verify from a tailnet device on the LAN

From Gundabad (with Tailscale installed) or your phone on Wi-Fi:

```bash
tailscale status
# barazinbar should be listed; note its 100.x.y.z address

ping 10.28.99.50        # pelargir via the advertised route
```

On the client, ensure "Use Tailscale subnets" / "Accept routes" is enabled (`tailscale up --accept-routes` on Linux clients; automatic on mobile).

### 6. Verify the connection is direct, not relayed

This is the test that justifies ADR-004's entire decision:

```bash
tailscale ping barazinbar
```

Required: `pong from barazinbar ... via <IP>:<port>` — a direct WireGuard path.
Bad: `via DERP(...)` — relayed. If it stays on DERP after several pings, check that pfSense isn't blocking UDP 41641 outbound and that outbound NAT isn't rewriting source ports unpredictably (this is the EIM-NAT issue from ADR-004 — it should not affect Barazinbar as a Linux client, which is the whole point of Barazinbar).

### 7. The real-world test

From your phone, **off Wi-Fi (cellular)**, Tailscale connected:

1. Open Jellyfin app → server `http://10.28.99.50:8096`
2. Sign in, play something
3. Watch `intel_gpu_top` on pelargir — remote playback that needs transcoding should light the Video engine exactly as the LAN test did

If playback works on cellular, the travel use case is proven.

### 8. Snapshot

```
barazinbar → Shutdown → Snapshots → "tailscale-working" → Start
```

Powered-off snapshot, per the Step 1 caveat.

---

## Security Notes

- Barazinbar is now a bridge between the internet (via tailnet) and VLAN 99. Its attack surface is the tailnet ACLs plus this VM's SSH. Keep it minimal: no other services land on this box, ever.
- Default tailnet ACLs allow all tailnet devices to reach all advertised routes. Fine for a personal tailnet; revisit ACLs if the tailnet ever gains users beyond you.
- No ports were opened on Khazad-dûm at any point. Verify: pfSense → Firewall → NAT — no port forwards referencing .50 or .51.

---

## k8s Migration Path (recorded for Phase 3)

When Jellyfin moves into the cluster, this subnet router is replaced by the **Tailscale Kubernetes operator** — services get exposed on the tailnet directly via operator-managed proxies, and Barazinbar retires. The tailnet itself (devices, auth, ACLs) carries over unchanged. Trigger to reevaluate earlier: pfSense CE 2.9 shipping stable EIM-NAT (see ADR-004, 90-day review window).

---

## Verification

- [ ] `sysctl net.ipv4.ip_forward` = 1
- [ ] `tailscale status` on barazinbar shows the node up
- [ ] Route 10.28.99.0/24 approved in admin console
- [ ] Key expiry disabled for barazinbar
- [ ] LAN device on tailnet can ping 10.28.99.50 through the route
- [ ] `tailscale ping barazinbar` shows a direct path, not DERP
- [ ] Jellyfin playback works from cellular with hardware transcoding active
- [ ] No port forwards on Khazad-dûm
- [ ] Powered-off snapshot taken

---

## What I Observed

*Fill in during the build.*

---

## What I Learned

*Fill in during the build.*

---

## Issues Encountered

| Issue | Cause | Fix |
|-------|-------|-----|
| | | |

---

*Phase 2 complete. Next phase: hardening (reverse proxy, TLS, container update policy) → see README build phases.*
