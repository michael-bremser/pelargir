# Build Journal — Step 2: iGPU Passthrough to pelargir VM

**Date:** —  
**Host:** Nogrod (10.28.99.11) for Parts 1–2; pelargir VM for Part 3  
**Status:** [ ] In Progress

---

## Objective

Pass the Intel UHD 630 from the Nogrod Proxmox host through to the pelargir VM via VFIO. Verify VA-API hardware acceleration inside the VM before anything else is installed on it.

**Prerequisite:** Step 1 complete, including a confirmed clean reboot of pelargir with OVMF/q35/host CPU and the GRUB EFI fix applied. If pelargir hasn't survived a clean boot yet, stop — go back. The entire diagnostic strategy of this step depends on the VM being known-good before the GPU touches it.

---

## Why This Order Matters

Host-side work first (Part 1), then attach the device (Part 2), then verify in the guest (Part 3). The GPU is the **last** thing added because a VM that boots cleanly without the GPU and hangs with it gives you a one-variable diagnosis. A VM built with the GPU attached from day one gives you nothing.

See ADR-002 for the reasoning behind VFIO passthrough vs alternatives.

---

## Part 1 — Proxmox Host (Nogrod)

SSH into Nogrod directly (10.28.99.11), not the pelargir VM.

### 1. Verify IOMMU is active

```bash
dmesg | grep -e DMAR -e IOMMU
```

Required line (typically the **last** line of output):

```
DMAR: Intel(R) Virtualization Technology for Directed I/O
```

> The output will also contain a block of `IOMMU feature ... inconsistent` lines on this hardware. **These are normal** — they refer to optional features that differ between the two DMAR units on Comet Lake. They are not errors. The only line that matters is the Directed I/O confirmation.

If that line is missing, enable `VT-d` in the Nogrod BIOS before continuing.

### 2. Find the iGPU and audio device IDs

```bash
lspci -nn | grep VGA
```

Expected (verified on this hardware):

```
00:02.0 VGA compatible controller [0300]: Intel Corporation CometLake-S GT2 [UHD Graphics 630] [8086:9bc8] (rev 03)
```

Note the PCI address (`00:02.0`) and device ID (`8086:9bc8`).

```bash
lspci -nn | grep -i audio
```

Note the Intel HD Audio device ID (e.g. `8086:06c8`). Use whatever your output shows.

### 3. Verify the IOMMU group

```bash
for d in /sys/kernel/iommu_groups/*/devices/*; do n=${d#*/iommu_groups/*}; n=${n%%/*}; printf 'IOMMU Group %s ' "$n"; lspci -nns "${d##*/}"; done | grep -i "00:02"
```

Verified result on this hardware: the UHD 630 sits alone in **IOMMU Group 0**. Clean isolation, no ACS workarounds needed. If your output ever shows it sharing a group with the NIC or storage, stop and research before passing anything through.

### 4. Enable IOMMU in GRUB

```bash
sudo vim /etc/default/grub
```

Change:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
```

to:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
```

Then:

```bash
update-grub
```

### 5. Load VFIO kernel modules

```bash
sudo vim /etc/modules
```

Append:

```
vfio
vfio_iommu_type1
vfio_pci
```

### 6. Bind the iGPU to VFIO and blacklist host drivers

Create the VFIO config — substitute your actual IDs from step 2:

```bash
sudo vim /etc/modprobe.d/vfio.conf
```

```
options vfio-pci ids=8086:9bc8,8086:06c8 disable_vga=1
```

> `disable_vga=1` disables VGA arbitration for the device. Per the Proxmox PCI passthrough documentation, this reduces the legacy VGA boot code in play under OVMF — it is part of why the UEFI path boots where SeaBIOS hung.

Blacklist the host drivers so they can't claim the device before VFIO:

```bash
sudo vim /etc/modprobe.d/blacklist-i915.conf
```

```
blacklist i915
blacklist snd_hda_intel
```

Apply:

```bash
update-initramfs -u -k all
```

> **Expected noise:** this prints a warning per installed kernel about a removable bootloader (`Removable bootloader found at '/boot/efi/EFI/BOOT/BOOTX64.efi'...`). That warning concerns the *Proxmox host's* bootloader and is informational on this machine — the initramfs images still generate successfully. It may also appear to stall after the last kernel; give it time to return to a prompt rather than Ctrl+C'ing it mid-write.

### 7. Reboot Nogrod

```bash
sudo reboot
```

> **This restarts every VM on Nogrod** — k3s-control, pelargir, and barazinbar all go down and come back (all are set `onboot: 1`). k3s tolerates this; the cluster reconverges when k3s-control returns. Schedule accordingly if anything on the finai cluster is mid-task.

### 8. Verify VFIO claimed the iGPU

After reboot:

```bash
lspci -nnk -s 00:02.0
```

Required:

```
Kernel driver in use: vfio-pci
```

If it shows `i915`, the blacklist didn't take — recheck both files in `/etc/modprobe.d/`, rerun `update-initramfs -u -k all`, reboot again. **Do not proceed to Part 2 until this shows vfio-pci.** Attaching a device the host still owns is how VMs hang at boot.

---

## Part 2 — Attach the GPU (Proxmox UI)

### 9. Shut down pelargir fully

```
pelargir → Shutdown
```

Full shutdown, not reboot — PCI device changes require a cold start.

### 10. Add the PCI device

```
pelargir → Hardware → Add → PCI Device → Raw Device
```

- **Device:** `0000:00:02.0` (Intel UHD 630)
- **All Functions:** checked (carries the audio function with it)
- **PCI-Express:** checked (this is why the VM is q35)
- **ROM-Bar:** checked (default)
- **Primary GPU:** **unchecked** — the virtual display stays primary so the Proxmox console keeps working; the iGPU is for transcoding only, it never drives a display

### 11. Cold boot and watch the console

```
pelargir → Start → Console
```

Expected: normal boot to login prompt. The line `snd_hda_intel ... no codecs found!` may appear — that's the passed-through audio function with nothing attached to it. Harmless.

**If the VM hangs at boot with the GPU attached** (it should not, on OVMF — but if it does):

1. Force stop: `pelargir → Stop`
2. Remove the PCI device: `Hardware → PCI Device → Remove`
3. Start — the VM boots normally again; nothing is lost
4. Diagnose from the known-good state: re-verify Part 1 step 8 on the host, confirm BIOS=OVMF and Machine=q35 in the VM hardware tab, then re-attach and retry

This remove-to-recover loop is safe and repeatable. The VM is never damaged by a failed GPU attach.

### 12. Check networking survived the PCI change

Adding a PCI device can shift the virtual PCI topology and rename the network interface (see Step 1, "Network Interface Naming"). If SSH to .50 fails after boot:

```bash
# On the console:
ip link show          # find the current interface name
```

If you applied the MAC-match netplan config in Step 1, the interface is pinned to `eth0` and this cannot happen. If you didn't, update `/etc/netplan/50-cloud-init.yaml` with the new name and `sudo netplan apply` — then go apply the MAC-match config so this is the last time.

---

## Part 3 — Inside the pelargir VM

SSH into pelargir (10.28.99.50).

### 13. Confirm the iGPU is visible

```bash
lspci -nnk | grep -A3 -E 'VGA|Display'
```

You should now see **two** display entries: the virtual console adapter (`1234:1111`, driver `bochs-drm`) and the **Intel UHD 630 (`8086:9bc8`)**. If only the virtual adapter appears, the passthrough isn't attached — back to Part 2.

Check the i915 driver bound to it inside the guest:

```bash
lspci -nnk -s $(lspci | grep -i 'UHD' | cut -d' ' -f1)
# "Kernel driver in use: i915"
```

### 14. Confirm the render node exists

```bash
ls -l /dev/dri/
```

Expected: `card0` (or `card1`) and **`renderD128`**. If `renderD128` is missing, reboot the VM once — i915 sometimes needs a clean initialization pass. Still missing after that: check `dmesg | grep i915` for errors.

### 15. Install Intel media drivers and verify VA-API

```bash
sudo apt update
sudo apt install -y intel-media-va-driver intel-gpu-tools vainfo
vainfo
```

Success looks like a `VAProfile` list including `VAProfileH264*` and `VAProfileHEVC*` entries with `VAEntrypointVLD` (decode) and `VAEntrypointEncSlice*` (encode). This is the proof Quick Sync is alive inside the VM.

If `vainfo` errors, point it at the device explicitly:

```bash
vainfo --display drm --device /dev/dri/renderD128
```

### 16. Record the render/video GIDs for Step 4

```bash
getent group render
getent group video
```

Write both numeric GIDs into the table below — the Docker compose file in Step 4 needs them, and they vary between installs (render is often 992 on Ubuntu 24.04, video is often 44 — use what *this* VM reports).

| Group | GID on pelargir |
|-------|-----------------|
| render | |
| video | |

### 17. Re-snapshot

The `pre-gpu` snapshot from Step 1 predates the working passthrough. Take a new **powered-off** snapshot:

```
pelargir → Shutdown → Snapshots → Take Snapshot → "gpu-working" → Start
```

---

## Verification

- [ ] Nogrod: `dmesg` shows Directed I/O enabled
- [ ] Nogrod: `lspci -nnk -s 00:02.0` shows `vfio-pci` in use
- [ ] pelargir hardware tab: PCI device 0000:00:02.0 attached, All Functions + PCIe, Primary GPU unchecked
- [ ] pelargir boots clean to login with GPU attached
- [ ] SSH to 10.28.99.50 works (interface name survived)
- [ ] `lspci` in VM shows UHD 630 with driver `i915`
- [ ] `/dev/dri/renderD128` exists
- [ ] `vainfo` lists H264 + HEVC profiles, no errors
- [ ] render/video GIDs recorded
- [ ] Powered-off `gpu-working` snapshot taken

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
| VM hangs at `grub-common.service` with GPU attached (SeaBIOS build) | GPU passthrough to SeaBIOS VM — no UEFI init path for the device | Rebuild VM as OVMF/q35 (Step 1); GPU passthrough requires UEFI |
| `update-initramfs` prints removable bootloader warnings | Proxmox host uses removable EFI path | Informational on the host; images generate fine |
| Networking dead after attaching GPU | PCI topology shift renamed the interface | MAC-match netplan config (Step 1) prevents permanently |

---

## Next Step

[Step 3 — NFS Mounts from Aglarond](step-03-nfs-mounts.md)
