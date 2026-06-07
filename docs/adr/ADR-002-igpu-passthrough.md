# ADR-002: Intel iGPU Passthrough for Quick Sync Transcoding

**Status:** Accepted  
**Date:** 2026-06-07  
**Author:** Mike

---

## Context

Jellyfin supports multiple transcoding modes. The Nogrod host has an i3-10100T with an Intel UHD 630 integrated GPU. Quick Sync is Intel's fixed-function hardware video transcoding engine — it handles H.264, H.265/HEVC, and AV1 decode/encode at a fraction of the CPU cost of software transcoding. Making Quick Sync available inside the jellyfin VM requires exposing the iGPU to the VM.

The decision is how to expose the iGPU to the VM, and what that means for the k8s migration path.

---

## Decision

**Pass the Intel UHD 630 through to the jellyfin VM via VFIO. Inside the VM, expose `/dev/dri` to the Jellyfin Docker container.**

---

## Options Considered

### Option A — Software Transcoding Only

Run Jellyfin with CPU-only transcoding. No iGPU involvement.

**Pros:**
- Zero passthrough configuration — works out of the box
- No risk of destabilizing the Proxmox host

**Cons:**
- The i3-10100T has 4 cores shared with the VM, the Proxmox host, and Barazinbar — software transcoding a single 4K HEVC stream can saturate all of them
- Concurrent streams are not viable under software transcoding on this hardware
- Quick Sync is purpose-built for this workload and sits idle — wasteful

### Option B — VA-API via /dev/dri in LXC

Expose the iGPU render node directly into an LXC container.

**Pros:**
- No VFIO required — simpler at the hypervisor level

**Cons:**
- Rejected at ADR-001 — LXC is not the chosen runtime
- Device passthrough to LXC is fragile and not the path forward

### Option C — VFIO iGPU Passthrough to VM (chosen)

Bind the Intel UHD 630 to the VFIO driver on the Proxmox host. Pass the PCI device through to the jellyfin VM. Inside the VM, install Intel media drivers. Docker container accesses `/dev/dri/renderD128` via a device mount.

**Pros:**
- The VM gets exclusive, direct hardware access to the iGPU — no virtualization overhead in the render path
- Standard, well-documented VFIO path on Proxmox
- Inside the VM, the iGPU looks like native hardware — all Intel media driver tooling works normally
- The Docker → k8s migration path is clean: in Kubernetes, the Intel GPU device plugin exposes `/dev/dri` to pods in exactly the same way — `resources.limits` with `gpu.intel.com/i915`. The Docker device mount and the k8s resource request are conceptually identical operations at different layers

**Cons:**
- Once the iGPU is bound to VFIO, the Proxmox host loses access to it — not relevant on a headless server with no display output
- Requires IOMMU enabled in BIOS and Proxmox kernel parameters (`intel_iommu=on`) — a one-time configuration step
- The i3-10100T iGPU shares an IOMMU group with other PCI devices depending on the platform — verify grouping before passthrough to avoid passing through unintended devices

---

## Reasoning

Quick Sync exists precisely for this workload. Leaving it idle while the CPU strains through software transcoding is the wrong tradeoff. The i3-10100T's Quick Sync implementation handles H.264 and HEVC encode/decode efficiently, and Jellyfin's VA-API integration with Intel hardware is mature and well-tested.

The VFIO path is more complex to configure than software transcoding, but it is a one-time setup. The operational cost after configuration is zero — the iGPU is just available to the VM.

The k8s migration story is direct. The Intel GPU device plugin for Kubernetes works via the same `/dev/dri` device nodes that Docker uses. The resource model is different (k8s uses `resources.limits` rather than `--device` flags) but the underlying mechanism is identical. Understanding how the device gets exposed in Docker makes the Kubernetes configuration legible rather than magical.

---

## IOMMU Group Note

Before configuring passthrough, verify the UHD 630's IOMMU group on Nogrod:

```bash
# Run on the Proxmox host
for d in /sys/kernel/iommu_groups/*/devices/*; do
  n=${d#*/iommu_groups/*}; n=${n%%/*}
  printf 'IOMMU Group %s ' "$n"
  lspci -nns "${d##*/}"
done | grep -i vga
```

If the iGPU shares a group with critical devices (e.g. the primary NIC), passthrough requires ACS override patches or a different approach. On most M70q units the iGPU is in its own group — confirm before proceeding.

---

## k8s Migration Path

| Docker Phase | Kubernetes Phase |
|---|---|
| `--device /dev/dri/renderD128` in compose | `resources.limits: gpu.intel.com/i915: "1"` in pod spec |
| Intel media drivers in VM | Intel GPU device plugin DaemonSet in cluster |
| VM has exclusive iGPU access | Node has iGPU, plugin advertises it to scheduler |

The VM-level passthrough is unchanged in the k8s phase. The only difference is who asks for the device — Docker vs the kubelet via the device plugin API.

---

## Consequences

- IOMMU enabled in Nogrod BIOS (`VT-d` / Intel Virtualization for Directed I/O)
- `intel_iommu=on iommu=pt` added to Proxmox host GRUB kernel parameters
- Intel UHD 630 bound to `vfio-pci` driver on Proxmox host
- PCI device passed through to jellyfin VM in Proxmox config
- `intel-media-va-driver` and `vainfo` installed in jellyfin VM to verify VA-API
- Jellyfin Docker container mounted with `/dev/dri:/dev/dri`
- Jellyfin configured to use VA-API hardware acceleration
- Proxmox host has no display output from iGPU after passthrough — headless operation unaffected
