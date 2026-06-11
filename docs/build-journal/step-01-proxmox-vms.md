# Build Journal — Step 1: Provision Proxmox VMs

**Date:** —  
**Host:** Nogrod (10.28.99.11)  
**Status:** [ ] In Progress

---

## Objective

Provision two VMs on Nogrod: `pelargir` for the media server workload and `barazinbar` for the Tailscale subnet router. Both VMs live on VLAN 99 alongside the finai cluster nodes.

This step covers VM creation and base OS configuration only. iGPU passthrough is Step 2 — the GPU is added to the VM **after** a confirmed clean boot, never during creation.

---

## Why Two VMs

One VM runs Jellyfin. One VM runs Tailscale. Each does one job.

The alternative is running Tailscale inside the pelargir VM. That works, but it creates an unnecessary dependency — if the Jellyfin container crashes or the VM needs maintenance, remote access goes down with it. Barazinbar is a separate, minimal VM whose only job is to stay connected to the tailnet.

See ADR-001 (VM over LXC) and ADR-004 (Tailscale via Barazinbar) for the reasoning behind each choice.

---

## VM Specifications

| VM | Host | vCPU | CPU Type | RAM | Disk | BIOS | Machine | OS | IP | Purpose |
|----|------|------|----------|-----|------|------|---------|----|----|---------|
| pelargir | Nogrod | 4 | **host** | 8GB | 32GB | **OVMF (UEFI)** | **q35** | Ubuntu 24.04 LTS | 10.28.99.50 (DHCP res.) | Jellyfin + Docker |
| barazinbar | Nogrod | 1 | x86-64-v2-AES | **2GB** | 8GB | SeaBIOS | i440fx | Ubuntu 24.04 LTS | 10.28.99.51 (DHCP res.) | Tailscale subnet router |

**Why these specs are not negotiable:**

- **OVMF (UEFI) on pelargir** — GPU passthrough requires UEFI firmware. A GPU passed to a SeaBIOS VM has no UEFI initialization path and the VM hangs at boot (specifically at `grub-common.service`). This was learned the hard way. Proxmox's own documentation states best passthrough compatibility is q35 + OVMF + PCIe.
- **q35 on pelargir** — required for PCIe passthrough (i440fx only does legacy PCI).
- **CPU type `host` on pelargir** — exposes the real CPU to the guest, required for the guest's media driver stack to correctly detect Quick Sync capabilities.
- **2GB RAM on barazinbar** — the Ubuntu 24.04 live installer crashes with "generating crash report" loops below ~1GB RAM. 512MB is enough to *run* tailscaled but not enough to *install* Ubuntu. 2GB avoids the problem entirely and Nogrod has the headroom.
- **Barazinbar stays SeaBIOS/i440fx** — it has no passthrough requirements; defaults are fine.

---

## Steps

### 1. ISO Location

The Ubuntu 24.04 LTS server ISO is stored on Aglarond. When selecting the ISO during VM creation, choose Aglarond from the storage dropdown.

### 2. Create the pelargir VM

```
Datacenter → Nogrod → Create VM
```

**General tab:**
- Name: `pelargir`

**OS tab:**
- Ubuntu 24.04 ISO from Aglarond storage

**System tab — this is where the passthrough-critical settings live:**
- BIOS: **OVMF (UEFI)**
- Add EFI Disk: **checked**, storage: local-lvm, pre-enrolled keys: checked
- Machine: **q35**
- SCSI Controller: VirtIO SCSI single

**Disks tab:**
- 32GB, storage: local-lvm, bus: VirtIO Block

**CPU tab:**
- Cores: 4
- Type: **host**

**Memory tab:**
- 8192 MB, ballooning: **off** (Memory/Minimum memory equal, or untick ballooning)

**Network tab:**
- Bridge: vmbr0, VLAN tag: 99, Model: VirtIO
- **MAC address: set manually if rebuilding** — if a DHCP reservation already exists for a previous pelargir VM, reuse that MAC here so the reservation continues to work. New build: leave auto, note the generated MAC for the reservation in step 4.

> **Do not add the iGPU PCI device during creation.** The GPU is added in Step 2 only after the VM has a confirmed clean boot. Adding it before the OS exists makes boot failures impossible to diagnose — you can't tell a GPU hang from an install problem.

### 3. Create the barazinbar VM

```
Datacenter → Nogrod → Create VM
```

- Name: `barazinbar`
- OS: Ubuntu 24.04 ISO from Aglarond
- System: leave defaults (SeaBIOS, i440fx)
- Disk: 8GB, local-lvm, VirtIO Block
- CPU: 1 core
- RAM: **2048 MB** — no ballooning. Do not be tempted to go lower; see spec table note. After install completes you may reduce to 1024 MB if you want the RAM back, but tailscaled is so light it's not worth the trouble.
- Network: VirtIO, vmbr0, VLAN tag: 99

### 4. Create DHCP reservations on Khazad-dûm

Get the MAC address for each VM:

```
VM → Hardware → Network Device → MAC address
```

In pfSense, create a static mapping for each MAC on VLAN 99:

```
Services → DHCP Server → VLAN99 → Static Mappings → Add
```

- pelargir → 10.28.99.50
- barazinbar → 10.28.99.51

Hostname matching the VM name. **Save and Apply.**

> **Lease conflict warning (learned the hard way):** if a VM boots *before* its reservation exists — or boots repeatedly during failed installs — pfSense issues a dynamic lease from the pool (e.g. .100). pfSense then honors that active lease over the reservation until the lease expires, which can be hours. `netplan apply`, `dhclient`, and rebooting the VM will NOT fix this — the server side is the problem, not the client.
>
> **The fix:** in pfSense go to `Status → DHCP Leases` and use the **"Clear all leases"** button at the bottom of the page, then reboot the VM. Individual lease deletion is not offered for active leases in the UI, and the dhcpd service cannot be restarted by the commands you'd expect on pfSense CE. Clear-all is safe: every device with a reservation or static IP comes back to its correct address.

### 5. Install Ubuntu on each VM

Start each VM and open the console. Ubuntu Server install — same process for both:

- Language: English
- **Installer update prompt:** if offered an installer update, choose **"Continue without updating."** The update check is a network operation that can hang indefinitely.
- Network: leave as DHCP — the reservation handles addressing. Verify on this screen that the VM shows its reserved IP (.50 / .51). If it shows a pool address (.100+), stop, fix the lease per the warning in step 4, and restart the install.
- **Mirror test:** if the installer stalls at "testing mirror locations," wait it out or select Done without waiting — it proceeds with the default mirror. Do not reset the VM mid-install; a reset here leaves a half-installed disk that boots to a dead console.
- Storage: use entire disk, LVM
- Profile: username + password, hostname matches VM name (`pelargir` / `barazinbar`)
- **Enable OpenSSH server: yes**
- Snaps: skip everything

After install completes, **remove the ISO** (`Hardware → CD/DVD → Edit → Do not use any media`) so future boots can't accidentally re-enter the installer.

### 6. Post-install: fix LVM allocation (both VMs)

Ubuntu's LVM install reserves unallocated space by default. Known issue across every Ubuntu VM in this lab:

```bash
df -h
# If / shows significantly less than the allocated disk:
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv && sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
df -h
```

### 7. Post-install: fix the GRUB EFI sync warning (pelargir only)

On UEFI VMs, the first `update-initramfs` run will print a warning about a removable bootloader GRUB isn't configured to update, and `grub-common.service` can hang at boot. Fix it now, before it costs a boot cycle:

```bash
echo 'grub-efi-amd64 grub2/force_efi_extra_removable boolean true' | sudo debconf-set-selections && sudo apt install --reinstall grub-efi-amd64
```

### 8. Post-install: base configuration (both VMs)

```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Verify hostname
hostnamectl

# Disable swap (consistent with lab practice)
sudo swapoff -a && sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Enable IP forwarding (required on barazinbar for subnet routing; harmless on pelargir)
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-forwarding.conf
sudo sysctl --system
```

### 9. Reboot each VM once and confirm a clean boot

```bash
sudo reboot
```

Watch the console. The VM must reach the login prompt with no hangs. **Do not proceed to Step 2 until pelargir survives a clean reboot.** This is the baseline that makes Step 2 diagnosable — after this, any new boot failure has exactly one suspect: the GPU.

### 10. Take snapshots

```
Proxmox UI → VM → Snapshots → Take Snapshot
Name: pre-gpu (pelargir) / pre-tailscale (barazinbar)
```

> **Snapshot caveat (learned the hard way):** a snapshot taken with `vmstate` (RAM included) captures the *running machine type*. Rolling back to a RAM snapshot taken under a different machine type/firmware than the current config boots the old machine type and silently breaks things — including renaming the network interface. Take snapshots **powered off** (no vmstate) for restore points that respect the current config.

---

## Network Interface Naming — Read Before Step 2

The network interface name inside the VM is derived from the virtual PCI topology. It changes when the machine type changes **and when PCI devices are added** — which is exactly what Step 2 does.

- On q35, the interface will be something like `enp6s18` (it was `ens18` on i440fx)
- Adding the GPU in Step 2 can shift the PCI layout and rename it again
- When networking dies after a hardware change, it is almost always this

The fix when it happens:

```bash
ip link show                # find the new interface name
sudo vim /etc/netplan/50-cloud-init.yaml   # update the interface name
sudo netplan apply
```

> The netplan file on Ubuntu 24.04 server installs is **`50-cloud-init.yaml`** — not `00-installer-config.yaml` as many guides claim.

A more robust option — match by MAC instead of name so renames never break networking again. Edit `/etc/netplan/50-cloud-init.yaml`:

```yaml
network:
  version: 2
  ethernets:
    primary:
      match:
        macaddress: "bc:24:11:xx:xx:xx"   # this VM's MAC
      set-name: eth0
      dhcp4: true
```

Then `sudo netplan apply`. The interface is now always `eth0` regardless of PCI topology. **Recommended on pelargir before Step 2.**

---

## Verification

- [ ] DHCP reservations created and **both VMs confirmed on .50 / .51** (not pool addresses)
- [ ] Both VMs reachable via SSH from Gundabad
- [ ] pelargir: BIOS shows OVMF, Machine shows q35, CPU type host (`VM → Hardware`)
- [ ] ISO detached from both VMs
- [ ] LVM using full disk on both (`df -h`)
- [ ] GRUB EFI fix applied on pelargir
- [ ] Swap disabled on both (`free -h`)
- [ ] IP forwarding enabled on both (`sysctl net.ipv4.ip_forward`)
- [ ] netplan MAC-match config applied on pelargir (recommended)
- [ ] Both VMs survive a clean reboot to login prompt
- [ ] Powered-off snapshots taken on both VMs

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
| Installer crash loop on barazinbar ("generating crash report") | 512MB RAM insufficient for Ubuntu 24.04 installer | Raise VM to 2048MB |
| Installer hangs at mirror test | Slow/blocked path to Ubuntu mirrors | "Continue without updating"; never reset mid-install |
| VM stuck on dynamic lease (.100) instead of reservation | Lease issued before reservation existed; pfSense honors active lease | pfSense → Status → DHCP Leases → **Clear all leases**, reboot VM |
| Boot hang at `grub-common.service` | Removable EFI bootloader not managed by GRUB packages | `force_efi_extra_removable` debconf + reinstall grub-efi-amd64 |
| Networking dead after machine type change | Interface renamed (`ens18` → `enp6s18`) | Update `50-cloud-init.yaml`; better: MAC-match + `set-name` |

---

## Next Step

[Step 2 — iGPU Passthrough to pelargir VM](step-02-igpu-passthrough.md)
