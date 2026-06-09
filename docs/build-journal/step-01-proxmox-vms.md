# Build Journal — Step 1: Provision Proxmox VMs

**Date:** 06/08/26  
**Host:** Nogrod (10.28.99.11)  
**Status:** [X] Complete 

---

## Objective

Provision two VMs on Nogrod: `jellyfin` for the media server workload and `barazinbar` for the Tailscale subnet router. Both VMs live on VLAN 99 alongside the finai cluster nodes.

This step covers VM creation and base OS configuration only. iGPU passthrough to the jellyfin VM is handled in Step 2 — do not attempt to configure it here.

---

## Why Two VMs

One VM runs Jellyfin. One VM runs Tailscale. Each does one job.

The alternative is running Tailscale inside the jellyfin VM. That works, but it creates an unnecessary dependency — if the Jellyfin container crashes or the VM needs maintenance, remote access goes down with it. Barazinbar is a separate, minimal VM whose only job is to stay connected to the tailnet. It has nothing to fail except `tailscaled`.

See ADR-001 (VM over LXC) and ADR-004 (Tailscale via Barazinbar) for the reasoning behind each choice.

---

## VM Specifications

| VM | Host | vCPU | RAM | Disk | OS | IP | Purpose |
|----|------|------|-----|------|----|----|---------|
| jellyfin | Nogrod | 4 | 8GB | 32GB | Ubuntu 24.04 LTS | DHCP reservation | Jellyfin + Docker |
| barazinbar | Nogrod | 1 | 512MB | 8GB | Ubuntu 24.04 LTS | DHCP reservation | Tailscale subnet router |

> **IP addresses:** managed via DHCP reservation on Khazad-dûm, not configured inside the VMs. After each VM is provisioned and boots, get its MAC address from the Proxmox network tab, then create a reservation in pfSense (Services → DHCP Server → VLAN 99) before proceeding. Note the assigned IPs here once reserved.

---

## Steps

### 1. ISO Location

The Ubuntu 24.04 LTS server ISO is stored on Aglarond. When selecting the ISO during VM creation, choose Aglarond from the storage dropdown.

### 2. Create the jellyfin VM

```
Datacenter → Nogrod → Create VM
```

Settings:
- **Name:** `pelegir`
- **OS:** Ubuntu 24.04 ISO from Aglarond storage
- **System:** leave defaults (BIOS: SeaBIOS, SCSI controller: VirtIO SCSI)
- **Disk:** 32GB, storage: local-lvm, bus: VirtIO
- **CPU:** 4 cores
- **RAM:** 8192 MB (8GB) — no ballooning
- **Network:** VirtIO, bridge: vmbr0, VLAN tag: 99

> **Do not add the iGPU PCI device yet.** That is Step 2. Attempting passthrough before the OS is installed and the VFIO driver is bound will cause the VM to fail to start.

### 3. Create the barazinbar VM

```
Datacenter → Nogrod → Create VM
```

Settings:
- **Name:** `barazinbar`
- **OS:** Ubuntu 24.04 ISO from Aglarond storage
- **System:** leave defaults
- **Disk:** 8GB, storage: local-lvm, bus: VirtIO
- **CPU:** 1 core
- **RAM:** 512 MB — no ballooning
- **Network:** VirtIO, bridge: vmbr0, VLAN tag: 99

### 4. Create DHCP reservations on Khazad-dûm

Before booting into the Ubuntu installer, get the MAC address for each VM from the Proxmox network tab:

```
VM → Hardware → Network Device → MAC address
```

Then in pfSense create a reservation for each MAC on VLAN 99:

```
Services → DHCP Server → VLAN99 → Static Mappings → Add
```

Set a hostname matching the VM name (`pelegir` / `barazinbar`) and pick two unused addresses in the 10.28.99.x range. The finai cluster uses .40 and .41 — choose something clearly separated (e.g. .50 and .51). Note the assigned IPs here before proceeding.

### 5. Install Ubuntu on each VM

Start each VM and open the console. Ubuntu Server install — same process for both:

- Language: English
- Network: leave as DHCP — the reservation on Khazad-dûm handles the rest
- Storage: use entire disk, LVM
- Profile: set username and password, hostname matches VM name (`pelegir` / `barazinbar`)
- **Enable OpenSSH server: yes**
- Snaps: skip everything

After install, verify each VM got its reserved IP:

```bash
ip addr show
```

### 6. Post-install: check and fix LVM allocation

Ubuntu's LVM installer does not use the full allocated disk by default. Check and fix on both VMs:

```bash
df -h
# If / shows significantly less than the allocated disk size:

sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv && sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
df -h
# Should now show full disk size
```

This is the same issue encountered in the finai cluster VMs. Check proactively — it is easier to fix now than after software is installed.

### 7. Post-install: base configuration

Run on both VMs:

```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Verify hostname
hostnamectl

# Disable swap (not required for Docker/Tailscale but consistent with lab practice)
sudo swapoff -a && sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Enable IP forwarding (required for Tailscale subnet routing on barazinbar; harmless on jellyfin)
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-forwarding.conf
sudo sysctl --system
```

### 8. Take snapshots

Before proceeding to any software installation, snapshot both VMs:

```
Proxmox UI → VM → Snapshots → Take Snapshot
Name: pre-install
```

---

## Verification

- [x] DHCP reservations created on Khazad-dûm for both VMs
- [x] Both VMs reachable via SSH from Gundabad
- [x] `pelegir` at reserved IP, `barazinbar` at reserved IP
- [x] Hostnames correct on both (`hostnamectl`)
- [x] LVM using full disk on both (`df -h`)
- [x] Swap disabled on both (`free -h`)
- [x] IP forwarding enabled on both (`sysctl net.ipv4.ip_forward`)
- [x] Snapshots taken on both VMs

---

## Next Step

[Step 2 — iGPU Passthrough to jellyfin VM](step-02-igpu-passthrough.md)
