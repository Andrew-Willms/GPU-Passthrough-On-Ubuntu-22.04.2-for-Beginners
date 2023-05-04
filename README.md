# GPU Passthrough with an Intel CPU and AMD GPU on Ubuntu 22.04.2 LTS Using IOMMU, QEMU, and Virtual Machine Manager.

_This guide was created on April 4, 2023 and was last updated on April 4, 2023. In case I forget to change the "last updated" date check the commit history._

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

&nbsp;

---

# 4. Determine Your Hardware IDs

&nbsp;

---

# 5. Configure Grub

&nbsp;

---

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
