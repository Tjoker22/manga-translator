# Proxmox VM Setup & GPU Passthrough Guide
# Ollama VM (VMID 380)

## Overview

This guide sets up a dedicated Ubuntu 24.04 VM on Proxmox (`genesis`) with full
PCIe passthrough of the NVIDIA RTX 2060. This VM runs **Ollama only** — it is the
GPU inference server for the lab. Other applications (such as the manga-translator)
run in separate VMs or containers and reach Ollama over the network.

Proxmox itself runs headless (no physical display) and is accessed via the web UI
at `https://192.168.0.152:8006`.

**VM identity (flat network phase):**
| Field | Value |
|---|---|
| VM Name | `ollama` |
| VMID | 380 |
| Flat IP | Set a DHCP reservation for this VM — record the MAC after creation and add to the flat network register |
| Final LAB IP | 192.168.20.80 (post-VLAN migration) |

> **Flat network note:** genesis and all its VMs are currently on `192.168.0.0/24`.
> The VM will receive a flat IP via DHCP. Set a reservation in your router/DHCP
> server once you have the VM's MAC address, and add it to `network/flat-network-settings-register.md`.

**Host hardware:**
- CPU: Ryzen 7 5700X (no integrated graphics)
- GPU: NVIDIA MSI GeForce RTX 2060 (6 GB VRAM) — compute capability 7.5 (exceeds the 5.0 minimum required by Ollama)
- RAM: 64 GB host / 32 GB allocated to VM

---

## Architecture Context

```
genesis (Proxmox host — 192.168.0.152)
└── VMID 380  ollama  (flat IP: DHCP reservation)
      └── Ollama service — listens on 0.0.0.0:11434
              ↑
    Accessed over the network by:
    - manga-translator VM / container
    - Any other app on the flat network
```

> **Why this split?**
> GPU passthrough is tied to a VM. Running Ollama in its own VM means the GPU
> resource is dedicated and cleanly isolated. Applications consuming Ollama are
> lightweight and do not need GPU access — they call the HTTP API exactly as they
> would a remote service. This also means the ollama VM can be snapshotted,
> upgraded, or rebuilt without touching application VMs.

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
> - `amd_iommu=on` — activates AMD's IOMMU hardware at kernel level
> - `iommu=pt` — "passthrough mode" — only activates IOMMU for devices being passed
>   through to VMs. Improves overall system performance.

---

### Step 3 — Load VFIO Kernel Modules

VFIO is the Linux kernel subsystem that handles device passthrough to VMs.
It intercepts the GPU from the host OS and hands it to the VM instead.

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

Look for:

```
Kernel driver in use: vfio-pci
```

If you see `vfio-pci` as the kernel driver, passthrough is ready.
If you see `nvidia` or `nouveau`, the blacklist did not apply — recheck Step 5.

---

## Phase 2 — Create the VM in Proxmox Web UI

Access the web UI at `https://192.168.0.152:8006`.

Upload Ubuntu 24.04 LTS ISO first:
`Datacenter → Storage → ISO Images → Upload`

Click **Create VM** and configure each tab:

### General Tab
| Setting | Value |
|---|---|
| Name | `ollama` |
| VM ID | `380` |

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
| Disk Size | 60 GB |
| Cache | Write back |
| Discard | checked (enables TRIM) |

> 60 GB is sufficient for the OS, Ollama, and model storage (Qwen2.5 7B is ~4.7 GB).
> Increase if you plan to pull multiple large models.

### CPU Tab
| Setting | Value |
|---|---|
| Cores | 8 |
| Type | `host` |

> **Why CPU type `host`?** This exposes the real CPU's feature flags to the VM.
> NVIDIA's drivers check for specific CPU features. Using `host` prevents
> compatibility issues and gives the VM full access to the Ryzen 5700X's capabilities.

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
| Bridge | `vmbr0` |

Click **Finish** — do not start the VM yet.

---

## Phase 3 — Add the GPU to the VM

In the Proxmox web UI:
`VMID 380 → Hardware → Add → PCI Device`

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
nano /etc/pve/qemu-server/380.conf
```

Add this line at the bottom:

```
args: -cpu 'host,+kvm_pv_unhalt,+kvm_pv_eoi,hv_vendor_id=NV43FIX,kvm=off'
```

> **What this does:** `hv_vendor_id=NV43FIX` sets a fake hypervisor vendor ID
> so the NVIDIA driver does not recognise it is running in a VM. `kvm=off`
> hides the KVM signature. Together these prevent the Code 43 error.

---

## Phase 4 — Install Ubuntu in the VM

Start the VM and open the Proxmox console to get through the installer.

During install:
- Choose **Minimized** installation (no desktop environment needed)
- Hostname: `ollama`
- Enable **OpenSSH server** (there is a checkbox during install)
- Allow updates during install if offered

After install completes and the VM reboots, note the IP address shown at login
or find it in your DHCP server. **Set a DHCP reservation for this IP immediately**
and record the MAC address in `network/flat-network-settings-register.md`.

SSH in from your main machine — you will not need the Proxmox console after this.

---

## Phase 5 — Install NVIDIA Drivers Inside the VM

```bash
# Update the system first
sudo apt update && sudo apt upgrade -y

# Add the graphics drivers PPA for latest stable drivers
sudo add-apt-repository ppa:graphics-drivers/ppa -y
sudo apt update

# Install driver (535 is current stable long-term branch for RTX 2060)
# RTX 2060 compute capability is 7.5 — meets the 5.0+ minimum required by Ollama
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

CUDA is NVIDIA's parallel computing platform. Ollama uses it to offload
inference from the CPU to the GPU.

```bash
sudo apt install -y nvidia-cuda-toolkit

# Verify
nvcc --version
```

If `nvcc` is not found after install:

```bash
export PATH=/usr/local/cuda/bin:$PATH
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
```

---

## Phase 7 — Install Ollama and Configure Network Binding

### Install Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Ollama installs as a systemd service and starts automatically. By default it
only binds to `127.0.0.1` — meaning nothing outside this VM can reach it.
Since the manga-translator (and any other client) runs in a separate VM, you
must change this.

### Configure OLLAMA_HOST for Network Access

Edit the Ollama systemd service to set the bind address:

```bash
sudo systemctl edit ollama
```

This opens a drop-in override file. Add:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

Save and apply:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

Verify Ollama is now listening on all interfaces:

```bash
sudo ss -tlnp | grep 11434
```

You should see `0.0.0.0:11434` rather than `127.0.0.1:11434`.

### Pull the Translation Model

```bash
# Qwen2.5 7B — fits entirely in the RTX 2060's 6 GB VRAM (~4.5 GB needed)
# Downloads approximately 4.7 GB
ollama pull qwen2.5:7b

# Test it
ollama run qwen2.5:7b "Translate from Japanese to English: ありがとうございます"
# Expected: "Thank you very much." or similar
```

> **How Ollama uses the GPU:**
> Ollama detects CUDA automatically after NVIDIA drivers are installed.
> For Qwen2.5 7B on the RTX 2060, the whole model fits in VRAM — every
> inference operation runs on the GPU.

**Optional — 14B model (slower, higher quality):**
```bash
ollama pull qwen2.5:14b
# ~8.5 GB — exceeds 6 GB VRAM. Ollama will offload overflow layers to system RAM.
# Will work but slower than 7B.
```

### Verify Remote Access

From another machine on the flat network, confirm the API is reachable:

```bash
curl http://<OLLAMA-VM-FLAT-IP>:11434
# Expected response: "Ollama is running"
```

> Replace `<OLLAMA-VM-FLAT-IP>` with the DHCP reservation you set.
> This is the address the manga-translator VM will point to.

---

## Snapshots Checklist

| Snapshot name | When to take it |
|---|---|
| `proxmox-pre-iommu` | Before editing any Proxmox host config |
| `ollama-ubuntu-fresh` | Right after Ubuntu installs, before any dev tools |
| `ollama-nvidia-ready` | After `nvidia-smi` confirms GPU is working |
| `ollama-ready` | After Ollama is running and reachable from the network |

---

## Quick Reference

| Task | Command |
|---|---|
| Check GPU | `nvidia-smi` |
| List pulled models | `ollama list` |
| Pull a model | `ollama pull qwen2.5:7b` |
| Test a model interactively | `ollama run qwen2.5:7b` |
| Check Ollama service status | `sudo systemctl status ollama` |
| Restart Ollama | `sudo systemctl restart ollama` |
| Check Ollama bind address | `sudo ss -tlnp \| grep 11434` |
| View Ollama environment config | `sudo systemctl cat ollama` |

---

## Troubleshooting

**`nvidia-smi` shows Code 43 / device not working:**
The NVIDIA driver detected the VM. Double-check the `args:` line in
`/etc/pve/qemu-server/380.conf`. The `hv_vendor_id` value must not contain
"kvm" or "proxmox" — use `NV43FIX` or any arbitrary string.

**VFIO not binding to GPU (still shows `nvidia` as kernel driver):**
The blacklist did not apply before the NVIDIA driver loaded. Confirm
`/etc/modprobe.d/blacklist.conf` has all three nvidia entries and rerun
`update-initramfs -u -k all`, then reboot.

**VM fails to start after adding PCI device:**
Check IOMMU groups on the Proxmox host:
```bash
find /sys/kernel/iommu_groups/ -type l | sort -V
```
The GPU and its audio device should be in the same IOMMU group. If other
devices share the group, you may need to pass them all through or use the
ACS override patch.

**Ollama not reachable from other VMs (connection refused):**
Confirm `OLLAMA_HOST=0.0.0.0:11434` is set in the systemd override and that
you reloaded and restarted the service. Run `sudo ss -tlnp | grep 11434`
to confirm the bind address.

**`nvcc --version` not found:**
```bash
export PATH=/usr/local/cuda/bin:$PATH
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
```
