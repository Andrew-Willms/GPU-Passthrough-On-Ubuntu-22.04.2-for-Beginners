# GPU Passthrough with an Intel CPU and AMD GPU on Ubuntu 22.04.2 LTS Using IOMMU, QEMU, and Virtual Machine Manager.

_This guide was created on April 4, 2023 and was last updated on April 4, 2023. In case I forget to change the "last updated" date check the commit history._

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

---

### Experience Level

I installed linux for the first time two days before writing these instructions. GPU passthrough can be hard but, hardware gods allowing, you too can enable high performance graphics in your virtual environment.

When I was getting GPU passthrough setup I found the guides that explained the goal of each step invaluable in troubleshooting so I will attempt to do the same. Since I am quite new to this my explanations may not be entirely correct or may be lacking; if you have a better explanation for something and would like to help improve this guide please let me know.

&nbsp;

---

### General Hardware Requirements

To perform GPU passthrough you must have a CPU, motherboard, and Bios that support IOMMU virtualization. You must also have two GPUs (one of these can be the integrated graphics on many CPUs). One GPU will display the graphics on the guest machine while the other (this is the one that can be an iGPU) will display the graphics for the host system.

If you only have one GPU GPU passthrough may still be possible; you should look into "Single GPU Passthrough".

My hardware setup:
- Intel i9-12900k
- Asus Prime Z690-P Wifi D4
- 64 GB of DDR4 4000 CL18
- Gigabyte Radeon RX 6650 XT

This guide contains the steps I used to enable GPU passthrough on my hardware, however I have attempted to include alternative nVidia and Intel instructions where applicable.

&nbsp;

---
