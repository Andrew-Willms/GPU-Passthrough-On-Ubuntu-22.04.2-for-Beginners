# GPU Passthrough with an Intel CPU and AMD GPU on Ubuntu 22.04.2 LTS Using IOMMU, QEMU, and Virtual Machine Manager.

_This guide was created on April 3, 2023 and was last updated on April 4, 2023. In case I forget to change the "last updated" date check the commit history._

&nbsp;

# 1. Table of Contents

| # | Sections |
| --- | ------------- |
| 1 | [Table of Contents](#1-table-of-contents) |
| 2 | [Introduction](#2-introduction) |
| 3 | [Bios Settings](#3-bios-settings) |
| 4 | [Determine Your Hardware IDs](#4-determine-your-hardware-ids) |
| 5 | [Configure Grub](#5-configure-grub) |
| 6 | [Configure VFIO](#6-configure-vfio) |
| 7 | [Setup Virtual Machine Manager](#7-setup-virtual-machine-manager) |
| 8 | [Setup Your Virtual Machine](#8-setup-your-virtual-machine) |
| 9 | [Conclusion](#9-conclusion) |

&nbsp;

---

# 2. Introduction

&nbsp;

### Acknowledgements
The information in this guide is primarly taken from [this answer](https://askubuntu.com/a/1410487/1692619) on [askubuntu.com](https://askubuntu.com/). 
Other resources this draws from are:
- https://mathiashueber.com/pci-passthrough-ubuntu-2004-virtual-machine/
- https://www.reddit.com/r/VFIO/comments/10umt32/error_43_with_a_passthrough_on_a_dual_amd_gpu/
- https://www.youtube.com/watch?v=KVDUs019IB8
- https://www.youtube.com/watch?v=eTX10QlFJ6c
- https://www.youtube.com/watch?v=jc3PjDX-CGs
- https://askubuntu.com/questions/1212969/softdep-for-vfio-pci-wont-work
- https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF

&nbsp;


### Experience Level

I installed linux for the first time two days before writing these instructions. GPU passthrough can be hard but, hardware gods allowing, you too can enable high performance graphics in your virtual environment.

When I was getting GPU passthrough setup I found the guides that explained the goal of each step invaluable in troubleshooting so I will attempt to do the same. Since I am quite new to this my explanations may not be entirely correct or may be lacking; if you have a better explanation for something and would like to help improve this guide please let me know.

&nbsp;


### General Hardware Requirements

To perform GPU passthrough you must have a CPU, motherboard, and Bios that support IOMMU virtualization (see [Bios Settings](#3-bios-settings) for details). You must also have two GPUs, one of these can be the integrated graphics found on many CPUs. One GPU will display the graphics for the guest system, while the other (this is the one that can be an iGPU) will display the graphics for the host system.

If you only have one GPU GPU passthrough may still be possible; you should look into "Single GPU Passthrough".

My hardware setup:
- Intel i9-12900k
- Asus Prime Z690-P Wifi D4
- 64 GB of DDR4 4000 CL18
- Gigabyte Radeon RX 6650 XT

This guide contains the steps I used to enable GPU passthrough on my hardware, however I have attempted to include alternative nVidia and Intel instructions where applicable.

&nbsp;

---

# 3. Bios Settings

IOMMU is the technology that will allow our virtual machine to connect directly to the GPU.

Enagle IOMMU in your bios. The exact settings that need to be change very depending on what platform you are using and the vendor of your motherboard vendor.

For Intel based systems enagle VT-x and VT-D. For AMD based systems enable SVM.

On my hardware (Asus Prime Z690-P Wifi D4) I had to enable

I also

Advanced Mode > PCAsomthing > IOMMU

&nbsp;

---

# 4. Determine Your Hardware IDs
In order to configure GPU passthrough you need to determine the PCI address(es) of your GPU and any other devices you wish to pass to your VM (though this guide will only focus on the GPU). This can be determined by examinging the output of the bash script shown bellow. (Disclaimer: I have never meaningfully used bash and so I cannot explain _how_ the script works beyond "Uh, it has for loops and it lists stuff").

<details>
  <summary>A note for linux newbies like myself:</summary>
  
  To run the bash script copy-paste it into your linux terminal (which can helpfully be opened by pressint `ctrl+alt+t`) and hitting enter. If you're new to terminals like I am, you don't paste into a terminal using `ctrl+v`. In Ubuntu's defauilt terminal you have to right click and select paste (hopefully there is a more convenient way because clicking is annoying).
</details>

```
shopt -s nullglob
for g in /sys/kernel/iommu_groups/*; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

<details>
  <summary>Example output from my system.</summary>
  
  ```
  IOMMU Group 0:
	  00:00.0 Host bridge [0600]: Intel Corporation Device [8086:4660] (rev 02)
  IOMMU Group 1:
  	00:01.0 PCI bridge [0604]: Intel Corporation 12th Gen Core Processor PCI Express x16 Controller #1 [8086:460d] (rev 02)
  IOMMU Group 10:
	  00:17.0 SATA controller [0106]: Intel Corporation Device [8086:7ae2] (rev 11)
  IOMMU Group 11:
  	00:1a.0 PCI bridge [0604]: Intel Corporation Device [8086:7ac8] (rev 11)
  IOMMU Group 12:
	  00:1b.0 PCI bridge [0604]: Intel Corporation Device [8086:7ac0] (rev 11)
  IOMMU Group 13:
	  00:1c.0 PCI bridge [0604]: Intel Corporation Device [8086:7ab8] (rev 11)
  IOMMU Group 14:
	  00:1c.2 PCI bridge [0604]: Intel Corporation Device [8086:7aba] (rev 11)
  IOMMU Group 15:
	  00:1d.0 PCI bridge [0604]: Intel Corporation Device [8086:7ab0] (rev 11)
  IOMMU Group 16:
	  00:1f.0 ISA bridge [0601]: Intel Corporation Device [8086:7a84] (rev 11)
	  00:1f.3 Audio device [0403]: Intel Corporation Device [8086:7ad0] (rev 11)
	  00:1f.4 SMBus [0c05]: Intel Corporation Device [8086:7aa3] (rev 11)
	  00:1f.5 Serial bus controller [0c80]: Intel Corporation Device [8086:7aa4] (rev 11)
  IOMMU Group 17:
	  01:00.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 10 XL Upstream Port of PCI Express Switch [1002:1478] (rev c1)
  IOMMU Group 18:
	  02:00.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 10 XL Downstream Port of PCI Express Switch [1002:1479]
  IOMMU Group 19:
	  03:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Device [1002:73ef] (rev c1)
  IOMMU Group 2:
	  00:02.0 VGA compatible controller [0300]: Intel Corporation AlderLake-S GT1 [8086:4680] (rev 0c)
  IOMMU Group 20:
	  03:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21 HDMI Audio [Radeon RX 6800/6800 XT / 6900 XT] [1002:ab28]
  IOMMU Group 21:
	  04:00.0 Non-Volatile memory controller [0108]: Sandisk Corp Device [15b7:5017] (rev 01)
  IOMMU Group 22:
	  05:00.0 Non-Volatile memory controller [0108]: Micron/Crucial Technology P2 NVMe PCIe SSD [c0a9:540a] (rev 01)
  IOMMU Group 23:
	  08:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8125 2.5GbE Controller [10ec:8125] (rev 05)
  IOMMU Group 3:
	  00:06.0 PCI bridge [0604]: Intel Corporation 12th Gen Core Processor PCI Express x4 Controller #0 [8086:464d] (rev 02)
  IOMMU Group 4:
	  00:0a.0 Signal processing controller [1180]: Intel Corporation Platform Monitoring Technology [8086:467d] (rev 01)
  IOMMU Group 5:
	  00:0e.0 RAID bus controller [0104]: Intel Corporation Volume Management Device NVMe RAID Controller [8086:467f]
  IOMMU Group 6:
	  00:14.0 USB controller [0c03]: Intel Corporation Device [8086:7ae0] (rev 11)
	  00:14.2 RAM memory [0500]: Intel Corporation Device [8086:7aa7] (rev 11)
  IOMMU Group 7:
	  00:14.3 Network controller [0280]: Intel Corporation Device [8086:7af0] (rev 11)
  IOMMU Group 8:
	  00:15.0 Serial bus controller [0c80]: Intel Corporation Device [8086:7acc] (rev 11)
	  00:15.1 Serial bus controller [0c80]: Intel Corporation Device [8086:7acd] (rev 11)
	  00:15.2 Serial bus controller [0c80]: Intel Corporation Device [8086:7ace] (rev 11)
  IOMMU Group 9:
	  00:16.0 Communication controller [0780]: Intel Corporation Device [8086:7ae8] (rev 11)
  ```
</details>

&nbsp;

This script prints out each IOMMU group in your system, the devices in those groups, and the PCI IDs of those devices. The PCI IDs of each device are listed in square brackets `[]` after the device name. Look for the devices(s) relating to the GPU you wish to pass through and take not of their PCI IDs.

For example the lines
```
IOMMU Group 19:
	03:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Device [1002:73ef] (rev c1)
...
IOMMU Group 20:
  03:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21 HDMI Audio [Radeon RX 6800/6800 XT / 6900 XT] [1002:ab28]
```
represent my GPU and my GPU's HDMI audio interface, respectively (both of which I, and probably you as well, want to pass to your VM). The IDs for these devices on my system are "1002:73ef" and "1002:ab28".

&nbsp;

Note: According to [this timestamp](https://youtu.be/jc3PjDX-CGs?t=220) of one of the Youtube videos linked above, if you wish to pass an IOMMU device to your VM you must pass through all the devices in the same IOMMU group. This can pose challenges if the device you wish to passthrough is not alone in its group (though it seems like GPUs are generally in their own group). Should you run into the a related problem, the linked video persents a work around.

&nbsp;

Other useful commands include:
- `lscpi` which lists the PCI devices in your system.
- `lcpci -nn` which lists the PCI devices in your system _and their PCI IDs_.
- `lscpi -nnk` which lists the PCI devices in your system, their PCI IDs, and some other information about them _including which driver they are using_ (which will be useful later).

&nbsp;

---

# 5. Configure Grub

_Get your grubby hands off (actually this is a better description for what is done in the next section but the subtitle works better here)._

&nbsp;

---

# 6. Configure VFIO

&nbsp;

---

# 7. Setup Virtual Machine Manager

&nbsp;

---

# 8. Setup Your Virtual Machine

&nbsp;

---

# 9. Conclusion

Hopefully it worked I guess?
