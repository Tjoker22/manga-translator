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

- Found tghat with the gpu set as the primary gpu for the machine you could not get a picture on the web ui consle and thus could not see the installer. Rooted and turned that setting off of rnow whilr installing. Will then turn the setting back on weh the install is complete and ssh connection is verified.
- Set a static ip for ollama vmid 380 of 192.168.0.180/24 with 192.168.0.153 pi-hole and 1.1.1.1 set as dns
- Installed live server with openssh server during install. 
- used defaults for msot all settings including disk size and layout
- sucessfull install of the server (ollama) and it is reacable via ssh to my created user

### Phase 5 - Install Nvidia drivers

- full system update and upgrade
- added graphics drives PPA for stabel nvidia drivers
`sudo add-apt-repository ppa:graphics-drivers/ppa -y
sudo apt update`
- updated the drive used in the guide from `535` to `570`
  - `sudo apt install -y nvidia-driver-570`
- **issue** with the driver being installed but not showing with `nvidia-smi`
  - `nvidia-driver-570 is already the newest version (570.211.01-0ubuntu1.24.04.1).`

```bash
~$ nvidia-smi
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver. Make sure that the latest NVIDIA driver is installed and running.
```

- made a few chnages and forced Secure Boot to disable with success and now have `nvidia-smi` displaying with driver `580`
  - planned on using `570` but aftert a couple remove/reinstalls i made the desicion to go forward with `580`. Seems that PPA may be pulling the newest driver

---

**Step 1** — From inside the VM, run:
`bashsudo mokutil --disable-validation`
It will ask you to set a one-time password — enter anything simple, you'll need to type it again on the next screen. Something like 12345678 is fine, it's only used once.

**Step 2** — Reboot the VM:
`bashsudo reboot`

**Step 3** — Watch the Proxmox console
Before Ubuntu boots you'll see a blue MOK Manager screen. Use the arrow keys to navigate:
Change Secure Boot state → enter your password → Continue → Yes → Reboot

**Step 4** — After reboot, verify:
`bashsudo mokutil --sb-state`
\# Should say: SecureBoot disabled
`sudo modprobe nvidia`
\# No output = success
`nvidia-smi`
\# Should show your RTX 2060

---

- look into this issue and also it appears that all ram is being used wehn lookin gat the vm staut page on the web ui, but when looking at btop i show that only approx 2% ram is in use?
  - since balloning was disabled during setup the vm has no way to accuratly display the memory useage with the proxmox host so it defaults to full use. 
  - running top or btop will show what the system is acctually using at any time so it would be advised to keep an extra window open for it
- 

### Phase 6 - Insall CUDA Toolkit

- used for parallel compute used to offload inference from cpu to gpu
- `sudo apt install -y nvidia-cuda-toolkit` for install and `nvcc --version` to verify

```bash
~$ nvcc --version
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2023 NVIDIA Corporation
Built on Fri_Jan__6_16:45:21_PST_2023
Cuda compilation tools, release 12.0, V12.0.140
Build cuda_12.0.r12.0/compiler.32267302_0
```

- version 12.0.140

### Phase 7 - Install Ollama and Config binding

- install ollama with:

```bash
curl -fsSL https://ollama.com/install.sh | sh`
```

- by default ollama will only listen to its local vm ip address 127.0.0.1
  - edit ollama systemd service to set bind address to 0.0.0.0 so all devices can connect.
  - `sudo systemctl edit ollama`
  - add this in the top blank space of proceeding file:

```bash
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

- save and then apply with `sudo systemctl daemon-reload && sudo systemctl restart ollama`
- verify ollama is listening on all interfaces:
- 
```bash
~$ sudo ss -tlnp | grep 11434
LISTEN 0      4096               *:11434            *:*    users:(("ollama",pid=3976,fd=3))
```

- *:11434 shows all interfaces are being heard
- pull ollama qwen2.5:7b model `ollama pull qwen2.5:7b`
- test with `ollama run qwen2.5:7b "Translate from Japanese to English: ありがとうございます"`
  - test was sucessful
  - verified remote access from admin machine 

```bash
❯ curl http://192.168.0.180:11434
Ollama is running%
```

- use the following to switch the model `ollama run qwen2.5:7b` or `ollama run qwen2.5:14b`
- issue with pulling the 14b model as there is no more spcae?
  - solved issue with the following:
  - 
```bash
run these to diagnose:
df -h
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              3.2G  1.3M  3.2G   1% /run
efivarfs                           256K   92K  160K  37% /sys/firmware/efi/efivars
/dev/mapper/ubuntu--vg-ubuntu--lv   28G   27G     0 100% /
tmpfs                               16G     0   16G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  105M  1.7G   6% /boot
/dev/sda1                          1.1G  6.2M  1.1G   1% /boot/efi
tmpfs                              3.2G   20K  3.2G   1% /run/user/1000
admin-yoyo@ollama:~$ du -sh ~/.ollama/models/
du: cannot access '/home/admin-yoyo/.ollama/models/': No such file or directory
admin-yoyo@ollama:~$ sudo du -sh ~/.ollama/models/
[sudo] password for admin-yoyo:
du: cannot access '/home/admin-yoyo/.ollama/models/': No such file or directory
admin-yoyo@ollama:~$ du -sh ~/.ollama/models/blobs/*
du: cannot access '/home/admin-yoyo/.ollama/models/blobs/*': No such file or directory
```

Classic Ubuntu LVM setup. The output tells us everything:

```
/dev/mapper/ubuntu--vg-ubuntu--lv   28G   27G   0   100%   /
```

**Suspect A confirmed.** Ubuntu installed with LVM and only carved out a 28 GB logical volume from your 60 GB disk. The other ~30 GB is sitting there unallocated. Also notice the error from the pull command told us exactly where Ollama stores its models — `/usr/share/ollama/.ollama/models/` — not your home directory. That's why your `du` commands found nothing.

---

## The Fix — Expand the LVM Logical Volume

This is a three-step process. No data loss, no reinstall. We're just telling Linux "use the space that's already there."

**Step 1 — Confirm the layout**
```bash
sudo pvs && sudo vgs && sudo lvs
```
This shows the physical volume, volume group, and logical volume sizes. Paste it if you want to double-check before proceeding, but the `df` output already makes the situation clear.
```shell
admin-yoyo@ollama:~$ sudo pvs && sudo vgs && sudo lvs
  PV         VG        Fmt  Attr PSize   PFree
  /dev/sda3  ubuntu-vg lvm2 a--  <56.95g 28.47g
  VG        #PV #LV #SN Attr   VSize   VFree
  ubuntu-vg   1   1   0 wz--n- <56.95g 28.47g
  LV        VG        Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  ubuntu-lv ubuntu-vg -wi-ao---- 28.47g
```

**Step 2 — Expand the physical volume to fill the disk**
```bash
sudo pvresize /dev/sda3
```
> `sda3` is almost certainly the LVM partition — Ubuntu's default installer puts EFI on sda1, boot on sda2, and LVM on sda3. The `pvresize` command tells LVM "the physical disk got bigger, update your records."
```shell
~$ sudo pvresize /dev/sda3 Physical volume "/dev/sda3" changed 1 physical volume(s) resized or updated / 0 physical volume(s) not resized
```

**Step 3 — Extend the logical volume into that free space**
```bash
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
```
> `-l +100%FREE` means "give this LV all remaining unallocated space in the volume group." You could also do `-L +30G` to add exactly 30 GB, but `+100%FREE` is cleaner.
```shell
~$ sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv Size of logical volume ubuntu-vg/ubuntu-lv changed from 28.47 GiB (7289 extents) to <56.95 GiB (14578 extents). Logical volume ubuntu-vg/ubuntu-lv successfully resized.
```

**Step 4 — Resize the filesystem to match**
```bash
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```
> The LV is now bigger, but the filesystem inside it doesn't know yet. `resize2fs` expands the ext4 filesystem to fill the new LV size. This is the step that actually makes the space usable.
```shell
~$ sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv 
resize2fs 1.47.0 (5-Feb-2023) Filesystem at /dev/mapper/ubuntu--vg-ubuntu--lv is mounted on /; on-line resizing required old_desc_blocks = 4, new_desc_blocks = 8 The filesystem on /dev/mapper/ubuntu--vg-ubuntu--lv is now 14927872 (4k) blocks long.
```

**Step 5 — Verify**
```bash
df -h /
```
You should now see something like:
```
/dev/mapper/ubuntu--vg-ubuntu--lv   57G   27G   28G   49%   /
```

```shell
~$ df -h / Filesystem Size Used Avail Use% Mounted on /dev/mapper/ubuntu--vg-ubuntu--lv 56G 27G 27G 50% /
```
--- 
