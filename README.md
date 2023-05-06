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

I installed linux for the first time two days before writing these instructions. GPU passthrough can be hard but, hardware gods permitting, you too can achieve high performance graphics in your virtual environment.

In general this guide will be targeted at those who are new to [GNU + Linux](https://www.reddit.com/r/copypasta/comments/63oudw/gnu_linux/).

When I was getting GPU passthrough setup, I found the guides that also explained the purpose of each step invaluable in troubleshooting so I will attempt to do the same. Since I am quite new to this my explanations may not be entirely correct or may be lacking; if you have a better explanation for something and would like to help improve this guide please let me know.

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


Advanced Mode > Advanced > CPU Configuration > Itel (VMX) Virtualizatoin Technology

Advanced Mode > Advanced > System Agent (SA) Configuration > VT-d: Enabled
                                                             Control Iommu Pre-boot Behavior: Enagle IOMMU during boot

Advanced Mode > Advanced\System Agent (SA) Configuration\Graphics Configuration > Primarry Display: CPU Graphics

check
Advanced Mode > Advanced > CPU Configuration > Intel VT-x Technology (Supported)

ReSize BAR (along the top bar) - off


&nbsp;

---

# 4. Determine Your Hardware IDs
In order to configure GPU passthrough you need to determine the PCI address(es) of your GPU and any other devices you wish to pass to your VM (though this guide will only focus on the GPU).

&nbsp;

1. These PCI address(es) can be determined by examinging the output of the bash script shown bellow. (Disclaimer: I have never meaningfully used bash and so I cannot explain _how_ the script works beyond "Uh, it has for loops and it lists stuff").

    <details>
        <summary>A note for linux newbies like myself:</summary>
  
        To run the bash script copy-paste it into your linux terminal (which can helpfully be opened by pressint `ctrl+alt+t`) and hitting enter. If you're new to terminals like I am, you don't paste into a terminal using `ctrl+v`. In Ubuntu's defauilt terminal you have to right click and select paste (hopefully there is a more convenient way because clicking is annoying).
    </details>

    ```bash
    shopt -s nullglob
    for g in /sys/kernel/iommu_groups/*; do
        echo "IOMMU Group ${g##*/}:"
        for d in $g/devices/*; do
            echo -e "\t$(lspci -nns ${d##*/})"
        done;
    done;
    ```

    <details>
        <summary>Example output from my system:</summary>
  
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
	
    </details>

    This script prints out each IOMMU group in your system, the devices in those groups, and the PCI IDs of those devices. The PCI IDs of each device are listed in square brackets `[]` after the device name. Look for the devices(s) relating to the GPU you wish to pass through and take not of their PCI IDs.

    For example the lines in my ouput  
    ```
    IOMMU Group 19:
	        03:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Device [1002:73ef] (rev c1)
    ...
    IOMMU Group 20:
            03:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21 HDMI Audio [Radeon RX 6800/6800 XT / 6900 XT] [1002:ab28]
    ```  
    represent my GPU and my GPU's HDMI audio interface, respectively (both of which I, and probably you as well, want to pass to your VM). The IDs for these devices on my system are "1002:73ef" and "1002:ab28".

    Note: According to [this timestamp](https://youtu.be/jc3PjDX-CGs?t=220) of one of the Youtube videos linked above, if you wish to pass an IOMMU device to your VM you must pass through all the devices in the same IOMMU group. This can pose challenges if the device you wish to passthrough is not alone in its group (though it seems like GPUs are generally in their own group). Should you run into the a related problem, the linked video persents a work around.

    Other useful commands include:
    - `lscpi` which lists the PCI devices in your system.
    - `lspci -n` which lists the IDs of the PCI devices in your system. (Okay I admit, this one doesn't seem that useful but I wanted to list it for completionism).
    - `lcpci -nn` which lists the PCI devices in your system _and their PCI IDs_.
    - `lscpi -nnk` which lists the PCI devices in your system, their PCI IDs, and some other information about them _including which driver they are using_ (which will be useful later).

&nbsp;

---

### Important Note:
(when you are done steps 5 and 6 you're host operating system should not be able to use the dgpu, it should be black on boot)

---
# 5. Configure Grub

_Get your grubby hands off (actually this is a better description for what is done in the next section but the subtitle works better here)._

What you actually do in this section is enable IOMMU in [grub](https://itsfoss.com/what-is-grub/) and tell grub what devices are going to be used for PCI passthrough.

&nbsp;

1. Run `sudo nano /etc/default/grub` to open up the grub config file. You can use `vim` or any other text editor instead of `nano` if you prefer.
    <details>
        <summary>The text file should look something like this:</summary>
	
	    # /boot/grub/grub.cfg.
        # For full documentation of the options in this file, see:
        #   info -f grub -n 'Simple configuration'

        GRUB_DEFAULT=0
        GRUB_TIMEOUT_STYLE=hidden
        GRUB_TIMEOUT=0
        GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
        GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
        GRUB_CMDLINE_LINUX=""

        # Uncomment to enable BadRAM filtering, modify to suit your needs
        # This works with Linux (no patch required) and with any kernel that obtains
        # the memory map information from GRUB (GNU Mach, kernel of FreeBSD ...)
        #GRUB_BADRAM="0x01234567,0xfefefefe,0x89abcdef,0xefefefef"

        # Uncomment to disable graphical terminal (grub-pc only)
        #GRUB_TERMINAL=console

        # The resolution used on graphical terminal
	
    </details>

&nbsp;

2. Change the line <br>
`GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"` to <br>
```GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on kvm.ignore_msrs=1 video=efifb:off vfio-pci.ids=1002:73ef,1002:ab28"```
    - `intel_iommu_on` enables IOMMU in grub. If you have an AMD system use `amd_iommu=on` instead.
    - `kvm.ignore_msrs=1` presumably makes something ignore something else (sorry, that's all I have).
    - `video=efifb:off` presumably tells grub not to use video drivers on the GPU to pass through. Not all guides included this bit but this is one of the steps that finally got it working for me and I haven't A/B tested to ensure this is indeed necessar.
    - `vfio-pci.ids=1002:73ef,1002:ab28` tells grub which devices you want to pass through. Again, not all guides included this but it was one of the steps that finally got GPU passthrough working for me.

&nbsp;

3. Save and exit the file.

&nbsp;

4. Execute `sudo grub-mkconfig -o /boot/grub/grub.cfg` to update the grub configuration.

    <details>
        <summary>The output should look something like this:</summary>
	
        Sourcing file `/etc/default/grub'
        Sourcing file `/etc/default/grub.d/init-select.cfg'
        Generating grub configuration file ...
        Found linux image: /boot/vmlinuz-5.19.0-41-generic
        Found initrd image: /boot/initrd.img-5.19.0-41-generic
        Found linux image: /boot/vmlinuz-5.19.0-32-generic
        Found initrd image: /boot/initrd.img-5.19.0-32-generic
        Memtest86+ needs a 16-bit boot, that is not available on EFI, exiting
        Warning: os-prober will not be executed to detect other bootable partitions.
        Systems on them will not be added to the GRUB boot configuration.
        Check GRUB_DISABLE_OS_PROBER documentation entry.
        Adding boot menu entry for UEFI Firmware Settings ...
        done
	
    </details>

&nbsp;

5. Reboot your system. (Hint: `sudo reboot` is likely the quickest way to do this).

    Once you have rebooted your system check that IOMMU is enabled by executing `sudo dmesg | grep -i -e DMAR -e IOMMU`.
    <details>
        <summary>The output should look something like this:</summary>
    
        [    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-5.19.0-41-generic root=UUID=3ba26aa1-311e-40b5-b3b4-5f95583c8364 ro quiet splash intel_iommu=on kvm.ignore_msrs=1 video=efifb:off vfio-pci.ids=1002:73ef,1002:ab28 vt.handoff=7
        [    0.006576] ACPI: DMAR 0x0000000072349000 000088 (v02 INTEL  EDK2     00000002      01000013)
        [    0.006611] ACPI: Reserving DMAR table memory at [mem 0x72349000-0x72349087]
        [    0.068845] Kernel command line: BOOT_IMAGE=/boot/vmlinuz-5.19.0-41-generic root=UUID=3ba26aa1-311e-40b5-b3b4-5f95583c8364 ro quiet splash intel_iommu=on kvm.ignore_msrs=1 video=efifb:off vfio-pci.ids=1002:73ef,1002:ab28 vt.handoff=7
        [    0.068878] DMAR: IOMMU enabled
        [    0.151435] DMAR: Host address width 39
        [    0.151436] DMAR: DRHD base: 0x000000fed90000 flags: 0x0
        [    0.151439] DMAR: dmar0: reg_base_addr fed90000 ver 4:0 cap 1c0000c40660462 ecap 29a00f0505e
        [    0.151440] DMAR: DRHD base: 0x000000fed91000 flags: 0x1
        [    0.151443] DMAR: dmar1: reg_base_addr fed91000 ver 5:0 cap d2008c40660462 ecap f050da
        [    0.151444] DMAR: RMRR base: 0x0000007c000000 end: 0x000000807fffff
        [    0.151446] DMAR-IR: IOAPIC id 2 under DRHD base  0xfed91000 IOMMU 1
        [    0.151446] DMAR-IR: HPET id 0 under DRHD base 0xfed91000
        [    0.151447] DMAR-IR: Queued invalidation will be enabled to support x2apic and Intr-remapping.
        [    0.152935] DMAR-IR: Enabled IRQ remapping in x2apic mode
        [    0.353359] pci 0000:00:02.0: DMAR: Skip IOMMU disabling for graphics
        [    0.375083] iommu: Default domain type: Translated 
        [    0.375083] iommu: DMA domain TLB invalidation policy: lazy mode 
        [    0.404051] DMAR: No ATSR found
        [    0.404052] DMAR: No SATC found
        [    0.404053] DMAR: IOMMU feature fl1gp_support inconsistent
        [    0.404053] DMAR: IOMMU feature pgsel_inv inconsistent
        [    0.404054] DMAR: IOMMU feature nwfs inconsistent
        [    0.404054] DMAR: IOMMU feature dit inconsistent
        [    0.404055] DMAR: IOMMU feature sc_support inconsistent
        [    0.404055] DMAR: IOMMU feature dev_iotlb_support inconsistent
        [    0.404056] DMAR: dmar0: Using Queued invalidation
        [    0.404057] DMAR: dmar1: Using Queued invalidation
        [    0.404465] pci 0000:00:00.0: Adding to iommu group 0
        [    0.404470] pci 0000:00:01.0: Adding to iommu group 1
        [    0.404474] pci 0000:00:02.0: Adding to iommu group 2
        [    0.404479] pci 0000:00:06.0: Adding to iommu group 3
        [    0.404483] pci 0000:00:0a.0: Adding to iommu group 4
        [    0.404487] pci 0000:00:0e.0: Adding to iommu group 5
        [    0.404494] pci 0000:00:14.0: Adding to iommu group 6
        [    0.404498] pci 0000:00:14.2: Adding to iommu group 6
        [    0.404502] pci 0000:00:14.3: Adding to iommu group 7
        [    0.404510] pci 0000:00:15.0: Adding to iommu group 8
        [    0.404514] pci 0000:00:15.1: Adding to iommu group 8
        [    0.404518] pci 0000:00:15.2: Adding to iommu group 8
        [    0.404524] pci 0000:00:16.0: Adding to iommu group 9
        [    0.404528] pci 0000:00:17.0: Adding to iommu group 10
        [    0.404540] pci 0000:00:1a.0: Adding to iommu group 11
        [    0.404545] pci 0000:00:1b.0: Adding to iommu group 12
        [    0.404559] pci 0000:00:1c.0: Adding to iommu group 13
        [    0.404565] pci 0000:00:1c.2: Adding to iommu group 14
        [    0.404570] pci 0000:00:1d.0: Adding to iommu group 15
        [    0.404581] pci 0000:00:1f.0: Adding to iommu group 16
        [    0.404585] pci 0000:00:1f.3: Adding to iommu group 16
        [    0.404589] pci 0000:00:1f.4: Adding to iommu group 16
        [    0.404594] pci 0000:00:1f.5: Adding to iommu group 16
        [    0.404599] pci 0000:01:00.0: Adding to iommu group 17
        [    0.404604] pci 0000:02:00.0: Adding to iommu group 18
        [    0.404615] pci 0000:03:00.0: Adding to iommu group 19
        [    0.404622] pci 0000:03:00.1: Adding to iommu group 20
        [    0.404626] pci 0000:04:00.0: Adding to iommu group 21
        [    0.404638] pci 0000:05:00.0: Adding to iommu group 22
        [    0.404643] pci 0000:08:00.0: Adding to iommu group 23
        [    0.406965] DMAR: Intel(R) Virtualization Technology for Directed I/O
        [    4.275811] AMD-Vi: AMD IOMMUv2 functionality not available on this system - This is not a bug.
    
    </details>

    After subsequent reboots of your system this command won't necessarily yield any output. My understanding is that this command basicaly checks a log file for instances of `"DMAR"` and `"IOMMU"` (grep is a command line formatting tool) and prints the relevant lines. The log file in question will probably only contains instances of `"DMAR"` and `"IOMMU"` after running `sudo grub-mkconfig -o /boot/grub/grub.cfg` and rebooting so running `sudo dmesg | grep -i -e DMAR -e IOMMU` on after subsequent reboots will likely yield no results.

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
