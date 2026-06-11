# Build Journal — Step 3.5: Barazinbar — Tailscale Subnet Router

**Date:** 2026-06-10  
**Node:** Barazinbar VM on Nogrod — SSH to 10.28.99.51  
**Status:** [ ] In Progress

---

## Objective

Configure Barazinbar — a minimal Ubuntu VM on Nogrod — as a Tailscale subnet router. Once complete, every device enrolled in your tailnet can reach both `10.28.99.0/24` (VLAN 99, homelab) and `10.28.20.0/24` (VLAN 20, trusted LAN) from anywhere. This enables remote kubectl, SSH to cluster nodes, and access to all management UIs (Proxmox, pfSense, TrueNAS) without opening any ports on Khazad-dûm.

---

## What's Actually Happening

Tailscale is a mesh VPN built on WireGuard. When two devices are enrolled in the same tailnet, they establish direct peer-to-peer encrypted tunnels — no traffic flows through a third-party server if a direct path is available.

Barazinbar is a **subnet router**: a node that advertises LAN subnets to the tailnet. When your laptop is enrolled and Barazinbar is advertising both subnets, your laptop routes traffic destined for either range through the WireGuard tunnel to Barazinbar, which forwards it onto the local network. From Barazinbar's perspective the traffic looks like it's coming from within the subnet — so you can SSH to `10.28.99.40` (k3s-control) or hit the Proxmox UI at `10.28.99.11:8006` exactly as if you were home. The same applies to any device on VLAN 20.

One VM, two subnet advertisements — every device on both VLANs is reachable. Adding new services requires no additional Tailscale configuration.

**Why not run Tailscale on Khazad-dûm (pfSense) directly:**  
pfSense CE 2.8 uses hard NAT (non-EIM), which forces Tailscale into DERP relay mode instead of direct peer-to-peer. The pfSense Tailscale package also has known subnet route instability on 2.7/2.8. A dedicated Linux VM avoids both issues. Reevaluate when pfSense CE 2.9 ships with confirmed EIM-NAT support.

---

## VM Specifications

| VM | Host | vCPU | RAM | Disk | OS | IP |
|----|------|------|-----|------|----|----|
| barazinbar | Nogrod | 1 | 2GB | 10GB | Ubuntu 24.04 LTS | 10.28.99.51 |

---

## Pre-flight Checks

- [x] VM provisioned on Nogrod — `barazinbar` at `10.28.99.51`
- [x] DHCP reservation set in pfSense for Barazinbar's MAC at `10.28.99.51`
- [ ] SSH access working from Gundabad (`ssh user@10.28.99.51`)
- [ ] You have a Tailscale account and access to the admin console at `login.tailscale.com`
- [ ] Inter-VLAN routing confirmed on Khazad-dûm — VLAN 99 can reach VLAN 20

---

## Step 1 — Post-Install Configuration

SSH in from Gundabad:

```bash
ssh user@10.28.99.51
```

Run the standard post-install baseline:

```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Verify hostname is correct
hostnamectl

# Enable IP forwarding — required for subnet routing
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/tailscale.conf
sudo sysctl -p /etc/sysctl.d/tailscale.conf

# Verify forwarding is on
sysctl net.ipv4.ip_forward
# Should return: net.ipv4.ip_forward = 1
```

Also check the LVM — Ubuntu's default allocator under-provisions the root volume. Fix it now before anything else:

```bash
df -h
# If root volume is not using the full 10GB:
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
df -h
```

---

## Step 2 — Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Verify the install:

```bash
tailscale version
sudo systemctl status tailscaled
```

---

## Step 3 — Bring Up Tailscale as Subnet Router

```bash
sudo tailscale up --advertise-routes=10.28.99.0/24,10.28.20.0/24 --advertise-exit-node
```

**Why each flag:**
- `--advertise-routes=10.28.99.0/24,10.28.20.0/24` — advertises both VLAN 99 (homelab) and VLAN 20 (trusted LAN) to the tailnet. Without this, Tailscale connects Barazinbar as an endpoint only — it won't forward subnet traffic
- `--advertise-exit-node` — makes Barazinbar available as an exit node, so your laptop can route all internet-bound traffic through your home network when remote. Your traffic egresses through Khazad-dûm exactly as if you were home, pfSense firewall rules and all

The command will print an authentication URL. Open it in a browser and log into your Tailscale account to authorize the node.

---

## Step 4 — Approve Routes in the Tailscale Admin Console

Tailscale requires subnet routes to be explicitly approved before they're active — deliberate safety gate.

1. Go to `login.tailscale.com` → Machines
2. Find `barazinbar` in the list
3. Click the three-dot menu → **Edit route settings**
4. Enable both routes: `10.28.99.0/24` and `10.28.20.0/24`
5. Approve the exit node under **Exit nodes**

---

## Step 5 — Configure Client Devices

On Gundabad (and any other device you want remote access from):

```bash
# If Tailscale isn't installed on Gundabad yet
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Accept the subnet routes
sudo tailscale up --accept-routes

# To use Barazinbar as an exit node when remote
sudo tailscale up --accept-routes --exit-node=barazinbar
```

On your phone or any other device: install the Tailscale app, log into your account, and enable the subnet route in the app settings.

---

## Step 6 — Snapshot

```
Proxmox UI → barazinbar VM → Snapshots → Take Snapshot
Name: post-tailscale-install
```

---

## Verification

Run these from outside your home network (phone hotspot works — make sure the exit node is off on your test device so traffic isn't looping back through the tunnel):

```bash
# Can you reach the cluster control plane?
kubectl get nodes

# Can you SSH to k3s-control?
ssh user@10.28.99.40

# Can you reach a VLAN 20 device? (e.g. Erebor Docker-Host)
ping 10.28.20.X

# Check Tailscale status on Barazinbar
ssh user@10.28.99.51
sudo tailscale status
```

Expected `tailscale status` output: Barazinbar listed as subnet router, your client device listed as connected, both routes showing.

Verification checklist:
- [ ] `kubectl get nodes` works from outside the network
- [ ] SSH to k3s-control (`10.28.99.40`) works from outside the network
- [ ] Proxmox UI reachable at `10.28.99.11:8006` from outside the network
- [ ] VLAN 20 device reachable from outside the network
- [ ] `tailscale status` on Barazinbar shows both subnet routes active
- [ ] Barazinbar shows up in Tailscale admin console as a subnet router with both routes approved

---

## What I Observed

*(Fill in during build)*

---

## What I Learned

*(Fill in during build)*

---

## Issues Encountered

| Issue | Cause | Fix |
|-------|-------|-----|
| | | |

---

## Security Notes

Barazinbar is a pivot point. If it were compromised, an attacker with tailnet access could reach everything on both VLANs. Hardening steps to address after the initial deployment:

- Disable password authentication for SSH — key-only login
- `ufw` or `nftables`: restrict inbound to SSH and Tailscale only
- Unattended upgrades for security patches
- Tailscale ACLs to restrict which tailnet devices can use the subnet routes

Not blocking for today — getting remote access working is the goal. Come back to hardening before this is your permanent setup.

---

## Future Reevaluation

If pfSense CE 2.9 ships with confirmed EIM-NAT support, evaluate migrating the subnet router role to Khazad-dûm and retiring Barazinbar. The tailnet itself (devices, ACLs, auth keys) does not change — only the mechanism bridging the subnets to it.

---

## Next Step

Step 4 — NVIDIA Device Plugin + Gundabad as bare metal GPU worker
