# Proxmox VM Setup & GPU Passthrough Guide

## Overview

This guide sets up a dedicated Ubuntu 24.04 development VM on Proxmox with full
PCIe passthrough of the NVIDIA RTX 2060. The GPU is passed entirely to the VM —
Proxmox itself runs headless (no physical display) and is accessed via the web UI
at `https://your-proxmox-ip:8006`.

**Host hardware:**
- CPU: Ryzen 7 5700X (no integrated graphics)
- GPU: NVIDIA MSI GeForce RTX 2060 (6 GB VRAM)
- RAM: 64 GB host / 32 GB allocated to VM

---

## Why GPU Passthrough

Running the manga translation pipeline without GPU passthrough means every
ML operation (manga-ocr, LaMa inpainting, Ollama LLM inference) runs on CPU only.
With the RTX 2060 available to the VM, all three stages use GPU acceleration,
making the pipeline 5–10x faster in practice.

---

## Phase 1 — Proxmox Host Configuration

> All steps in this phase run on the **Proxmox host** via SSH or the host shell.
> Take a snapshot of any existing VMs before starting.

### Step 1 — Enable IOMMU in BIOS

Reboot the Proxmox host and enter BIOS. Navigate to:

```
Advanced → AMD CBS → NBIO Common Options → IOMMU → Enabled
```

May also be labeled "AMD-Vi" depending on your motherboard manufacturer.
Save and boot back into Proxmox.

> **What is IOMMU?**
> IOMMU (Input-Output Memory Management Unit) is hardware built into the CPU and
> chipset that allows virtual machines to directly access physical hardware devices.
> Without it, the hypervisor has to emulate every hardware interaction in software,
> which is slow. With it, the VM talks to the real hardware directly and securely.

---

### Step 2 — Enable IOMMU in the Bootloader

```bash
nano /etc/default/grub
```

Find the line starting with `GRUB_CMDLINE_LINUX_DEFAULT` and edit it to:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"
```

Apply the change:

```bash
update-grub
```

> **What these parameters do:**
> - `amd_iommu=on` — activates AMD's IOMMU hardware at kernel level (Intel systems use `intel_iommu=on`)
> - `iommu=pt` — "passthrough mode" — only activates IOMMU for devices being passed through to VMs, not every device. Improves overall system performance.

---

### Step 3 — Load VFIO Kernel Modules

VFIO is the Linux kernel subsystem that handles device passthrough to VMs.
It acts as the middleman — intercepting the GPU from the host OS and handing
it to the VM instead.

```bash
sudo nano /etc/modules-load.d/vfio.conf
```

Add these four lines:

```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

---

### Step 4 — Find Your GPU's Hardware IDs

```bash
lspci -nn | grep -i nvidia
```

Expected output (your IDs will differ):

```
09:00.0 VGA compatible controller [0300]: NVIDIA Corporation GeForce RTX 2060 [10de:1e04]
09:00.1 Audio device [0403]: NVIDIA Corporation TU104 HD Audio Controller [10de:10f7]
```

**Copy both IDs** — the `[XXXX:XXXX]` values for both the video device and the
audio device. The audio controller is part of the same physical card and must
be passed through together.

---

### Step 5 — Bind VFIO to the GPU

Tell the kernel to give these devices to VFIO instead of the NVIDIA driver:

```bash
nano /etc/modprobe.d/vfio.conf
```

Add (using your actual IDs from Step 4):

```
options vfio-pci ids=10de:1e04,10de:10f7
```

Blacklist NVIDIA drivers on the host so they cannot claim the card before VFIO:

```bash
nano /etc/modprobe.d/blacklist.conf
```

Add:

```
blacklist nouveau
blacklist nvidia
blacklist nvidiafb
```

Update the initial ramdisk so these changes load at boot:

```bash
update-initramfs -u -k all
```

---

### Step 6 — Reboot and Verify

```bash
reboot
```

After reboot, SSH back in and confirm VFIO has control of the GPU:

```bash
lspci -nnk | grep -A3 nvidia
```

Look for this in the output:

```
Kernel driver in use: vfio-pci
```

If you see `vfio-pci` as the kernel driver, passthrough is ready.
If you see `nvidia` or `nouveau`, the blacklist did not apply — recheck Step 5.

---

## Phase 2 — Create the VM in Proxmox Web UI

Access the web UI at `https://your-proxmox-ip:8006`.

Upload Ubuntu 24.04 LTS ISO first:
`Datacenter → Storage → ISO Images → Upload`

Click **Create VM** and configure each tab:

### General Tab
| Setting | Value |
|---|---|
| Name | `manga-translator` |
| VM ID | next available |

### OS Tab
| Setting | Value |
|---|---|
| ISO Image | Ubuntu 24.04 LTS |

### System Tab — Critical for GPU passthrough
| Setting | Value |
|---|---|
| Machine | `q35` |
| BIOS | `OVMF (UEFI)` |
| Add EFI Disk | checked |
| SCSI Controller | `VirtIO SCSI single` |

> **Why q35 and OVMF?** PCIe passthrough requires a chipset that supports PCIe.
> The `q35` machine type emulates a modern Intel Q35 chipset with PCIe slots.
> OVMF is the UEFI firmware for VMs — modern NVIDIA drivers require UEFI boot.

### Disks Tab
| Setting | Value |
|---|---|
| Bus/Device | SCSI |
| Disk Size | 80 GB |
| Cache | Write back |
| Discard | checked (enables TRIM) |

### CPU Tab
| Setting | Value |
|---|---|
| Cores | 8 |
| Type | `host` |

> **Why CPU type `host`?** This exposes the real CPU's feature flags to the VM
> instead of a generic virtual CPU. NVIDIA's drivers check for specific CPU
> features. Using `host` type prevents compatibility issues.

### Memory Tab
| Setting | Value |
|---|---|
| Memory | 32768 MB (32 GB) |
| Ballooning Device | unchecked |

> Ballooning dynamically adjusts VM RAM. It can cause instability with GPU
> passthrough — disable it.

### Network Tab
| Setting | Value |
|---|---|
| Model | `VirtIO (paravirtualized)` |

Click **Finish** — do not start the VM yet.

---

## Phase 3 — Add the GPU to the VM

In the Proxmox web UI:
`Your VM → Hardware → Add → PCI Device`

| Setting | Value |
|---|---|
| Device | RTX 2060 (from dropdown) |
| All Functions | checked |
| ROM-Bar | checked |
| PCI-Express | checked |
| Primary GPU | checked |

### Apply the NVIDIA Code 43 Fix

NVIDIA drivers detect when running inside a VM and deliberately disable
themselves (known as "Error Code 43"). This fix disguises the VM.

Open the VM config file on the Proxmox host:

```bash
nano /etc/pve/qemu-server/YOUR-VM-ID.conf
```

Add this line at the bottom:

```
args: -cpu 'host,+kvm_pv_unhalt,+kvm_pv_eoi,hv_vendor_id=NV43FIX,kvm=off'
```

> **What this does:** `hv_vendor_id=NV43FIX` sets a fake hypervisor vendor ID
> so the NVIDIA driver does not recognise it is running in a VM. `kvm=off`
> hides the KVM signature. Together these prevent the Code 43 error.
> This is a well-documented, stable workaround used universally for NVIDIA passthrough.

---

## Phase 4 — Install Ubuntu in the VM

Start the VM and open the Proxmox console to get through the installer.

During install:
- Choose **Minimized** installation (no desktop environment needed)
- Enable **OpenSSH server** (there is a checkbox during install)
- Allow updates during install if offered

After install completes and the VM reboots, SSH in from your main machine.
You will not need the Proxmox console after this point.

---

## Phase 5 — Install NVIDIA Drivers Inside the VM

```bash
# Update the system first
sudo apt update && sudo apt upgrade -y

# Add the graphics drivers PPA for latest stable drivers
sudo add-apt-repository ppa:graphics-drivers/ppa -y
sudo apt update

# Install driver (535 is current stable long-term branch for RTX 2060)
sudo apt install -y nvidia-driver-535

# Reboot to load the driver
sudo reboot
```

After reboot, SSH back in and verify:

```bash
nvidia-smi
```

Expected output:

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.x       Driver Version: 535.x    CUDA Version: 12.x         |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
|   0  NVIDIA GeForce RTX 2060  Off  | 00000000:09:00.0 Off |                  N/A |
+-------------------------------+----------------------+----------------------+
```

Seeing your RTX 2060 listed here confirms passthrough is fully working.

---

## Phase 6 — Install CUDA Toolkit

CUDA is NVIDIA's parallel computing platform. Python libraries like PyTorch use
it to offload math from the CPU to the GPU. Without it, everything runs on CPU
even though the GPU is present and working.

```bash
sudo apt install -y nvidia-cuda-toolkit

# Verify
nvcc --version
```

---

## Snapshots Checklist

Take these snapshots in Proxmox at the indicated points. They are your safety net —
GPU passthrough is the most complex part of this setup and you want rollback points.

| Snapshot name | When to take it |
|---|---|
| `proxmox-pre-iommu` | Before editing any Proxmox host config |
| `ubuntu-fresh` | Right after Ubuntu installs, before any dev tools |
| `nvidia-ready` | After `nvidia-smi` confirms GPU is working |
| `dev-env-ready` | After all Python packages install successfully |

---

## Troubleshooting

**`nvidia-smi` shows Code 43 / device not working:**
The NVIDIA driver detected the VM. Double-check the `args:` line in the VM config
file. The `hv_vendor_id` value must not contain "kvm" or "proxmox" — use `NV43FIX`
or any arbitrary string.

**VFIO not binding to GPU (still shows `nvidia` as kernel driver):**
The blacklist did not apply before the NVIDIA driver loaded. Confirm
`/etc/modprobe.d/blacklist.conf` has all three nvidia entries and rerun
`update-initramfs -u -k all`, then reboot.

**VM fails to start after adding PCI device:**
Check that IOMMU groups are correct. Run on the Proxmox host:
```bash
find /sys/kernel/iommu_groups/ -type l | sort -V
```
The GPU and its audio device should be in the same IOMMU group. If other devices
share the group, you may need to pass them all through or use ACS override patch.

**`nvcc --version` not found after installing cuda toolkit:**
```bash
export PATH=/usr/local/cuda/bin:$PATH
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
```
