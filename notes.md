# Manga-Translator Notes

## Build

### gpu-passthrough

- ran some commands via cli to check and see if IOMMU was already enabled and running.
  - resuts found IOMMU is enabled, populated, and good to go
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
  
  - ultimately created a new file in the /etc/moduel-load.d/ called vfio.conf amd added the kernel modules from above. 
  - ran `update-initramfs -u -k all` for reboot persitence then rebooted the system
  - `lspci -nnk | grep -A3 nvidia ` after reboot

```bash
~$ lspci -nnk | grep -A3 nvidia
        Kernel modules: nvidiafb, nouveau, nova_core
2b:00.1 Audio device [0403]: NVIDIA Corporation TU104 HD Audio Controller [10de:10f8] (rev a1)
        Subsystem: Micro-Star International Co., Ltd. [MSI] Device [1462:3752]
        Kernel driver in use: snd_hda_intel
--
        Kernel driver in use: nvidia-gpu
        Kernel modules: i2c_nvidia_gpu
2c:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Function [1022:148a]
        Subsystem: Micro-Star International Co., Ltd. [MSI] Device [1462:7c56]
2d:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Reserved SPP [1022:1485]
```

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

- 