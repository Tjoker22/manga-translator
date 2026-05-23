# Manga-Translator Notes

## GPU-passthrough setup

### Phase 1 - Proxmox Host Configuration

- ran `dmesg | grep -e DMAR -e IOMMU -e AMD-Vi` and `ls /sys/kernel/iommu_groups` to check and see if IOMMU was already enabled and running.

```bash
~$ dmesg | grep -e DMAR -e IOMMU -e AMD-Vi
dmesg: read kernel buffer failed: Operation not permitted
~$ sudo dmesg | grep -e DMAR -e IOMMU -e AMD-Vi
[    0.162370] AMD-Vi: Using global IVHD EFR:0x0, EFR2:0x0
[    0.748642] pci 0000:00:00.2: AMD-Vi: IOMMU performance counters supported
[    0.751039] AMD-Vi: Extended features (0x58f77ef22294a5a, 0x0): PPR NX GT IA PC GA_vAPIC
[    0.751047] AMD-Vi: Interrupt remapping enabled
[    0.753162] perf/amd_iommu: Detected AMD IOMMU #0 (2 banks, 4 counters/bank).
~$ ls /sys/kernel/iommu_groups
0  1  10  11  12  13  14  15  16  17  18  19  2  3  4  5  6  7  8  9
```

  - resuts found IOMMU is enabled, populated, and good to go

- edit`/etc/default/grub'
  - edit `GRUB_CMDLINE_LINUX_DEFAULT` to `GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"`
  - activates amd_iommu hardware @ kernel lvl and iommu_pt enables "passthrough mode" only for devices being passed through to vms
- VFIO kernel modules
  - nano /etc/modules

```bash
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

  - come with this error as location of the file has moved:

```bash
# /etc/modules is obsolete and has been replaced by /etc/modules-load.d/.
# Please see modules-load.d(5) and modprobe.d(5) for details.
#
# Updating this file still works, but it is undocumented and unsupported.
```
  
  - created a new file in the /etc/moduel-load.d/ called vfio.conf amd added the kernel modules from above. 
  - updated this part of the setup guide

- find gpuid's with `~$ lspci -nn | grep -i nvidia`

```bash
~$ lspci -nn | grep -i nvidia
2b:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU104 [GeForce RTX 2060] [10de:1e89] (rev a1)
2b:00.1 Audio device [0403]: NVIDIA Corporation TU104 HD Audio Controller [10de:10f8] (rev a1)
2b:00.2 USB controller [0c03]: NVIDIA Corporation TU104 USB 3.1 Host Controller [10de:1ad8] (rev a1)
2b:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU104 USB Type-C UCSI Controller [10de:1ad9] (rev a1)
```

  - - VGA: `10de:1e89`  Audio: `10de:10f8`

- Bind VFIO to gpu at `nano /etc/modprobe.d/vfio.conf`
  - add `options vfio-pci ids=10de:1e89,10de:10f8`

- Blacklist NVIDIA drivers on host `nano /etc/modprobe.d/blacklist.conf`
```bash
blacklist nouveau
blacklist nvidia
blacklist nvidiafb
```

- run `update-initramfs -u -k all` to update for persitent settings after a reboot looking for `Kernel driver in use: vfio-pci`

```bash
~$ lspci -nnk | grep -A3 nvidia
        Kernel modules: nvidiafb, nouveau, nova_core
2b:00.1 Audio device [0403]: NVIDIA Corporation TU104 HD Audio Controller [10de:10f8] (rev a1)
        Subsystem: Micro-Star International Co., Ltd. [MSI] Device [1462:3752]
        Kernel driver in use: vfio-pci
--
        Kernel driver in use: nvidia-gpu
        Kernel modules: i2c_nvidia_gpu
2c:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Function [1022:148a]
        Subsystem: Micro-Star International Co., Ltd. [MSI] Device [1462:7c56]
2d:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Reserved SPP [1022:1485]
```

### Phase 2 - Create the VM in Proxmox Web UI

**Work done via the proxmox webui**

- Created the vm `ollama` with `vmid 380` with the following settings:
```bash
General Tab
| Setting | Value |
|---|---|
| Name | `ollama` |
| VM ID | `380` |

OS Tab
| Setting | Value |
|---|---|
| ISO Image | Ubuntu 24.04 LTS |

System Tab — Critical for GPU passthrough
| Setting | Value |
|---|---|
| Machine | `q35` |
| BIOS | `OVMF (UEFI)` |
| Add EFI Disk | checked |
| SCSI Controller | `VirtIO SCSI single` |

> **Why q35 and OVMF?** PCIe passthrough requires a chipset that supports PCIe.
> The `q35` machine type emulates a modern Intel Q35 chipset with PCIe slots.
> OVMF is the UEFI firmware for VMs — modern NVIDIA drivers require UEFI boot.

Disks Tab
| Setting | Value |
|---|---|
| Bus/Device | SCSI |
| Disk Size | 60 GB |
| Cache | Write back |
| Discard | checked (enables TRIM) |

> 60 GB is sufficient for the OS, Ollama, and model storage (Qwen2.5 7B is ~4.7 GB).
> Increase if you plan to pull multiple large models.

CPU Tab
| Setting | Value |
|---|---|
| Cores | 8 |
| Type | `host` |

> **Why CPU type `host`?** This exposes the real CPU's feature flags to the VM.
> NVIDIA's drivers check for specific CPU features. Using `host` prevents
> compatibility issues and gives the VM full access to the Ryzen 5700X's capabilities.

Memory Tab
| Setting | Value |
|---|---|
| Memory | 32768 MB (32 GB) |
| Ballooning Device | unchecked |

> Ballooning dynamically adjusts VM RAM. It can cause instability with GPU
> passthrough — disable it.

Network Tab
| Setting | Value |
|---|---|
| Model | `VirtIO (paravirtualized)` |
| Bridge | `vmbr0` |
```

### Phase 3 - Add gpu to vm

**as root in proxmox webui**

- `VMID 380 → Hardware → Add → PCI Device`
- from Raw Device dropdown list, find and select gpu
  - check all functions, rom-bar, pci-express, and primary gpu
- apply nvidia code 43 fix
  - `nano /etc/pve/qemu-server/380.conf`
  - add line at bottom: 
    - `args: -cpu 'host,+kvm_pv_unhalt,+kvm_pv_eoi,hv_vendor_id=NV43FIX,kvm=off'`

> **What this does:** `hv_vendor_id=NV43FIX` sets a fake hypervisor vendor ID
> so the NVIDIA driver does not recognise it is running in a VM. `kvm=off`
> hides the KVM signature. Together these prevent the Code 43 error.

### Phase 4 - Insatll Ubuintu in the VM
