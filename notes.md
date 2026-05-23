# Manga-Translator Notes

## Build

### gpu-passthrough

- ran some commands via cli to check and see if IOMMU was already enabled and running.
  - resuts found that it was enabled, populated, and good to go
```bash
admin-yoyo@genesis:~$ dmesg | grep -e DMAR -e IOMMU -e AMD-Vi
dmesg: read kernel buffer failed: Operation not permitted
admin-yoyo@genesis:~$ sudo dmesg | grep -e DMAR -e IOMMU -e AMD-Vi
[sudo] password for admin-yoyo:
[    0.162370] AMD-Vi: Using global IVHD EFR:0x0, EFR2:0x0
[    0.748642] pci 0000:00:00.2: AMD-Vi: IOMMU performance counters supported
[    0.751039] AMD-Vi: Extended features (0x58f77ef22294a5a, 0x0): PPR NX GT IA PC GA_vAPIC
[    0.751047] AMD-Vi: Interrupt remapping enabled
[    0.753162] perf/amd_iommu: Detected AMD IOMMU #0 (2 banks, 4 counters/bank).
admin-yoyo@genesis:~$ ls /sys/kernel/iommu_groups
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

