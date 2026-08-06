# GPU Passthrough On Ubuntu 22.04.2 for Beginners

**GPU passthrough with an Intel CPU, AMD GPU, and Asus Motherboard on Ubuntu 22.04.2 LTS. Instructions for other hardware are included but not tested.**

_This guide was created on May 3, 2023 and was last updated on July 7, 2026. In case I forget to change the "last updated" date check the commit history._

&nbsp;<br />
&nbsp;

# 1. Table of Contents

| # | Sections |
| --- | ------------- |
| 1 | [Table of Contents](#1-table-of-contents) |
| 2 | [Introduction](#2-introduction) |
| 3 | [Bios Settings](#3-bios-settings) |
| 4 | [Determine Your Hardware IDs](#4-determine-your-hardware-ids) |
| 5 | [Configure GRUB](#5-configure-grub) |
| 6 | [Configure VFIO](#6-configure-vfio) |
| 7 | [Setup Your Virtual Machine Manager](#7-setup-your-virtual-machine-manager) |
| 8 | [Setup Your Virtual Machine](#8-setup-your-virtual-machine) |
| 9 | [Trouble Shooting](#9-trouble-shooting) |


&nbsp;<br />
&nbsp;

# 2. Introduction

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

<div style="margin: 4em;"></div>

### Experience Level

I installed linux for the first time two days before writing these instructions. GPU passthrough can be hard, but hardware gods permitting, you too can achieve high performance graphics in your virtual environment.

In general, this guide will be targeted at those who are new to [GNU + Linux](https://www.reddit.com/r/copypasta/comments/63oudw/gnu_linux/).

When I was getting GPU passthrough setup, I found the guides that also explained the purpose of each step invaluable in troubleshooting so I will attempt to do the same. Since I am quite new to this, my explanations may not be entirely correct or may be lacking; if you have a better explanation for something and would like to help improve this guide, please do so. The [issues](https://github.com/Andrew-Willms/GPU-Passthrough-On-Ubuntu-22.04.2-for-Beginners/issues) are open.

<div style="margin: 4em;"></div>

### General Hardware Requirements

To perform GPU passthrough as described in this guide, you must have a CPU, motherboard, and Bios that support IOMMU virtualization (see [Bios Settings](#3-bios-settings) for details). Your system must also have two GPUs, at least one of which must be a dedicated GPU (the other can be the integrated graphics processor found on many CPUs). The dGPU will provide the graphics for the guest system, while the other will provide the graphics for the host system. If you wish to be able to view the host and guest systems simultaneously, you will also need at least two monitors.

<div style="margin: 4em;"></div>

### Single GPU Passthrough
If you only have one GPU, passthrough may still be possible but this guide will only be partly relevant to you. Instead you should look into "Single GPU Passthrough". In this configuration only one system gets a display adapter at a time and control can be passed between the two by executing a script on the host system.

<div style="margin: 4em;"></div>

### iGPU Passthrough *\[citations needed]*
It may be *possible* to pass an iGPU through to a guest system, but it will likely be much more difficult. Some of these difficulties may arise from:
- The iGPU not having dedicate VRAM and instead sharing system ram.
- The iGPU sharing other hardware (such as media encoders and decoders) with the rest of the CPU.
- Depending on the motherboard hardware and firmware, the iGPU may be initialized earlier, preventing the VFIO drivers from loading correctly.

Some modern Intel CPUs support Intel GVT-g and Intel SR-IOV, technologies that provide ways to pass part of an iGPU to a guest system.<br />
If you must pass an iGPU through to a guest system, good luck. You will likely need it.

<div style="margin: 4em;"></div>

### My Specific Hardware
- Intel i9-12900k
- Asus Prime Z690-P Wifi D4 (BIOS version 2212, release date 2022/12/13)
- 64 GB of DDR4 4000 CL18 *(oh, for RAM to be affordable again)*
- Gigabyte Radeon RX 6650 XT
- Ubuntu 22.04.2 LTS

This guide contains the steps I used to enable GPU passthrough on my hardware, however I have attempted to include alternative nVidia and Intel instructions where applicable.

&nbsp;<br />
&nbsp;

# 3. Bios Settings

[IOMMU](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit) (Input-Output Memory Management Unit) is the technology that will allow the virtual machine to connect directly to the GPU.

The exact settings required to enable IOMMU may vary depending on the platform of your system and the vendor of your motherboard. For Intel based systems enable VT-x and VT-D. For AMD based systems enable SVM. Many BIOSs have I search feature that may help find the appropriate settings.

In my BIOS (Asus Prime Z690-P Wifi D4, BIOS version 2212) I had to enable:
- `Advanced Mode > Advanced \ CPU Configuration \ Intel (VMX) Virtualizatoin Technology`
- `Advanced Mode > Advanced \ System Agent (SA) Configuration \ VT-d: Enabled`
- `Advanced Mode > Advanced \ System Agent (SA) Configuration \ Control Iommu Pre-boot Behavior: Enagle IOMMU during boot`
- `Advanced Mode > Advanced \ System Agent (SA) Configuration \ Graphics Configuration \ Primarry Display: CPU Graphics`

I also noted that `Advanced Mode > Advanced \ CPU Configuration \ Intel VT-x Technology` said `Supported`.

<div style="margin: 2em;"></div>

**Important Note:** In order to get my GPU working with my virtual machine, I had to turn off resizable bar ("ReSize BAR" in the top bar of my BIOS). I don't know if this step is required for all systems with support for resizable bar, but it was for mine.

&nbsp;<br />
&nbsp;

# 4. Determine Your Hardware IDs
In order to configure GPU passthrough, you need to determine the PCI address(es) of your GPU and any other devices you wish to pass to your VM (though this guide will only focus on the GPU).

These PCI address(es) can be determined by examinging the output of the bash script shown bellow. (Disclaimer: I have never meaningfully used bash and so I cannot explain _how_ the script works beyond "It has a for loops and prints stuff").

  
<div style="border: 0.17rem solid #000000; padding: 0.7rem; margin: 0.7rem 0rem; border-radius: 0.35rem;">

<details>
    <summary>A note for those new to terminals:</summary>

<div style="margin: 0.7rem;"></div>

To run the bash script copy-paste it into your terminal (which can helpfully be opened by pressing <kbd>Ctrl + Alt + T</kbd>) and hitting enter. To paste, don't press <kbd>Ctrl + V</kbd> (as you would in many other programs), instead press <kbd>Ctrl + Shift + V</kbd> or right click and select paste.
	
</details>

</div>

```bash
shopt -s nullglob
for g in /sys/kernel/iommu_groups/*; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

<div style="border: 0.17rem solid #000000; padding: 0.7rem; margin: 0.7rem 0rem; border-radius: 0.35rem;">

<details>
<summary>Example output from my system:</summary>

<div style="margin: 0.7rem;"></div>

<pre style="margin: 0px"><code>IOMMU Group 0:
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
        00:16.0 Communication controller [0780]: Intel Corporation Device [8086:7ae8] (rev 11)</code></pre>

</details>

</div>

This script prints out each IOMMU group in your system, the devices in those groups, and the PCI IDs of those devices. The PCI IDs of each device are listed in square brackets `[]` after the device name. Look for the devices(s) relating to the GPU you wish to pass through and take not of their PCI IDs.

For example the lines in my ouput show below represent my GPU and my GPU's HDMI audio interface, respectively (both of which I, and probably you as well, want to pass to your VM). The IDs for these devices on my system are `1002:73ef` and `1002:ab28`.
```
IOMMU Group 19:
        03:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Device [1002:73ef] (rev c1)
...
IOMMU Group 20:
        03:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21 HDMI Audio [Radeon RX 6800/6800 XT / 6900 XT] [1002:ab28]
```

Note: According to [this timestamp](https://youtu.be/jc3PjDX-CGs?t=220) of one of the Youtube videos linked above, if you wish to pass an IOMMU device to your VM, you must pass through all the devices in the same IOMMU group. This can pose challenges if the device you wish to passthrough is not alone in its group (or in a group exclusively with devices you want to pass through). Should you run into this problem, the video linked above persents a work around.

Note: According to [this part](https://askubuntu.com/questions/1406888/ubuntu-22-04-gpu-passthrough-qemu#:~:text=If%20You%20see%20text%20%22Kernel%20driver%20in%20use%3A%20nvidia%22%20like%20below%3A) of the main source of this guide, the proprietary nVidia drivers don't work with GPU passthrough and will have to be replaced by the Nouveau open source drivers. The link above provides some instruction for this. However, in other guides (for example [this one](https://www.youtube.com/watch?v=KVDUs019IB8)) the proprietary nVidia drivers are used.

Other potentially useful commands:
- `lscpi` which lists the PCI devices in your system.

- `lcpci -nn` which lists the PCI devices in your system and their PCI IDs.

- `lscpi -nnk` which lists the PCI devices in your system, their PCI IDs, and some other information about them, including which driver they are using (which will be useful later).

&nbsp;<br />
&nbsp;

# 5. Configure GRUB

_Get your grubby hands off (this is a better description for what is done in the next section but the pun works better here)._

[GRUB](https://itsfoss.com/what-is-grub/) (Grand Unified Bootloader) is the bootloader used by many popular Linux distros including Ubuntu. A bootloader is the program responsible for loading and starting your computers operating system after the hardware turns on. This section describes how to enable IOMMU in GRUB and tell GRUB what devices are going to be used for PCI passthrough. 

<div style="margin: 2em;"></div>

1. Run `sudo nano /etc/default/grub` to open up the grub config file.

    <div style="border: 0.17rem solid #000000; padding: 0.7rem; margin: 0.7rem 0rem; border-radius: 0.35rem;">

    <details>
        <summary>Explanation of <code>sudo nano /etc/default/grub</code>.</summary>
	
    <div style="margin: 0.7rem;"></div>

    - `sudo` is short for "superuser do". Starting a command with `sudo` means it is executed with root priviledges. Since we are editing system configuration we need root priviledges.

    - Nano is a text editor. Executing `nano` opens a file in Nano so we can edit it. You can use vim or any other text editor if you prefer.

    - `/etc/default/grub` is the path of the file we want to edit. The `/etc` directory stores system-wide configuration files. Intuitively, the `/etc/default` directory stores the default configuration files for our system. All together, `/etc/default/grub` is the path to the default configuration file for GRUB.
	
    </details>

    </div>

    <div style="border: 0.17rem solid #000000; padding: 0.7rem; margin: 0.7rem 0rem; border-radius: 0.35rem;">

    <details>
    <summary>The text file should look something like this:</summary>
	
    <div style="margin: 0.7rem;"></div>

    <pre style="margin: 0px"><code> # /boot/grub/grub.cfg.
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

    # The resolution used on graphical terminal</code></pre>

    </details>

    </div>

<div style="margin: 2em;"></div>

2. Change the line <br>
`GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"` to <br>
```GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on kvm.ignore_msrs=1 video=efifb:off vfio-pci.ids=1002:73ef,1002:ab28"```

    - `intel_iommu_on` enables IOMMU in GRUB. If you have an AMD CPU use `amd_iommu=on` instead.

    - `kvm.ignore_msrs=1` presumably makes something ignore something else (sorry, that's all I have).

    - `video=efifb:off` tells the Linux kernel not to use the "Extensible Firmware Interface Frame Buffer" (efifb) driver during boot. Without this, the host system may use the GPU while booting and VFIO (details in the [Configure VFIO](#6-configure-vfio) section) may not be able to claim the GPU later. Not all guides included this, but adding this was one of the steps that finally got everything working for me.

    - `vfio-pci.ids=1002:73ef,1002:ab28` tells GRUB which devices you want to reserve for the VFIO driver. Again, not all guides included this, but adding this was one of the steps that got everything working for me.

<div style="margin: 2em;"></div>

3. Save and exit the file. To do this press <kbd>Ctrl + X</kbd> to exit and <kbd>Y</kbd> to confirm you want to save the file. Nano helpfully displays shortcuts for commands you may want to use at the bottom.

<div style="margin: 2em;"></div>

4. Execute `sudo grub-mkconfig -o /boot/grub/grub.cfg` to scan for GRUB configuration changes, generate the new boot instructions, and write the output to `/boot/grub/grub.cfg`.

    <div style="border: 0.17rem solid #000000; padding: 0.7rem; margin: 0.7rem 0rem; border-radius: 0.35rem;">

    <details>
    <summary>The output should look something like this:</summary>

    <div style="margin: 0.7rem;"></div>

    <pre style="margin: 0px"><code> Sourcing file `/etc/default/grub'
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
    done</code></pre>
	
    </details>

    </div>

<div style="margin: 2em;"></div>

5. Reboot your system. (Hint: `sudo reboot` is likely the quickest way to do this).

<div style="margin: 2em;"></div>

6. Once you have rebooted your system, check that IOMMU is enabled by executing `sudo dmesg | grep -i -e IOMMU -e DMAR` if you have an Intel CPU or `sudo dmesg | grep -i -e IOMMU -e AMD-Vi -e IVRS` if you have an AMD CPU and consulting the output.

    <div style="border: 0.17rem solid #000000; padding: 0.7rem; margin: 0.7rem 0rem; border-radius: 0.35rem;">

    <details>
        <summary>Explanation of <code>sudo dmesg | grep -i -e ...</code></summary>
    
    <div style="margin: 0.7rem;"></div>

    - `dmsg` or "diagnostic message" is a command that outputs log messages generated by the Linux kernel.

    - The pipe operator `|` pipes the output of the previous command (`dmesg`) into the next command (`grep`).

    - `grep` is a command that searches through lines of text and returns the lines that match a given pattern. It is often used to filter large sets of log messages for only the messages that contain certain keywords.

    - `-i` is an option that makes the GREP search case insensitive ("i" for ignore case).

    - `-e` is an option indicating the next text is a pattern to search for.

    - `IOMMU`, `DMAR`, `AMD-Vi`, and `IVRS` are the text we are looking for. IOMMU has been explained before, DMAR stands for Direct Memory Access Remapping and is Intel's implementation of IOMMU, and AMD-Vi and IVRS are the AMD equivalent of DMAR.

    - Together, `sudo dmesg | grep -i -e ...` means look through all the kernel logs and return the ones relating to GPU passthrough and the settings we just changed.

    </details>

    </div>

    <div style="border: 0.17rem solid #000000; padding: 0.7rem; margin: 0.7rem 0rem; border-radius: 0.35rem;">

    <details>
    <summary>The output should look something like this (my output):</summary>
    
    <div style="margin: 0.7rem;"></div>

    <pre style="margin: 0px"><code> [    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-5.19.0-41-generic root=UUID=3ba26aa1-311e-40b5-b3b4-5f95583c8364 ro quiet splash intel_iommu=on kvm.ignore_msrs=1 video=efifb:off vfio-pci.ids=1002:73ef,1002:ab28 vt.handoff=7
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
    [    4.275811] AMD-Vi: AMD IOMMUv2 functionality not available on this system - This is not a bug.</code></pre>
    
    </details>

    </div>

    The GRUB configuration worked for my system, but I am not exactly sure which of these log messages indicated success. Consequently, this makes it difficult to say what lines you should check for in your output. In addition, some messages, such as `DMAR: No SATC found` and `DMAR: IOMMU feature fl1gp_support inconsistent`, seem line warnings or errors, yet evidently do not preclude success. My intuition is that the most important lines are as follows:

    - This line, which echos the parameters we entered in `/etc/default/grub`.

        ```
        [    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-5.19.0-41-generic root=UUID=3ba26aa1-311e-40b5-b3b4-5f95583c8364 ro quiet splash intel_iommu=on kvm.ignore_msrs=1 video=efifb:off vfio-pci.ids=1002:73ef,1002:ab28 vt.handoff=7
        ```

    - This line, for obvious reasons.

        ```
        [    0.068878] DMAR: IOMMU enabled
        ```

    - These lines, indicating devices (possibly CPU cores, though I am not sure) are being added to IOMMU groups. Strangely, I notice that non of the PCI IDs are the IDs I specifically passed through. I am not sure why this is, but again, everything worked for me.

        ```
        [    0.404465] pci 0000:XX:XX.X: Adding to iommu group X
        ```

    - And finally, this line (or an equivalent line with `AMD-Vi` for AMD CPUs).
        ```
        [    0.406965] DMAR: Intel(R) Virtualization Technology for Directed I/O
        ```

    I would guess that if your output contains lines similar to the above, and no lines explicitly indicating failure, then everything up this point has likely worked. I would also guess that if the command yields little or no output then configuring GRUB did not work.

    One additional note: My system only shows these messages on the first reboot after reconfiguring GRUB. If I reboot a second time and then execute `sudo dmesg | grep -i -e ...` there is no output. This indicates these logs are only created if the GRUB configuration has been changed, and are not generated during subsequent reboots. For this reason, you should configure GRUB, reboot, and check `dmseg` promptly. If you reboot again before checking `dmseg`, it may be harder to tell if the GRUB configuration worked.

&nbsp;<br />
&nbsp;

# 6. Configure VFIO
In this section we configure the [VFIO](https://docs.kernel.org/driver-api/vfio.html#:~:text=The%20VFIO%20driver%20is%20an,non%2Dprivileged%2C%20userspace%20drivers.) drivers. The VFIO drivers run on the host machine instead of the regular drivers and allow the guest machine to access the passed-through GPU.

<div style="margin: 2em;"></div>

1. Execute `sudo nano /etc/modprobe.d/vfio.conf`. `modprobe` is a utility used to load or unload modules from the Linux kernel. `/etc/modprobe.d` is a the directory where `modprobe` configuration is stored. By editing `/etc/modprobe.d/vfio.conf` we can dicate how and when the VFIO drivers are loaded. Most likely this configuration file does not yet exist, and so running this command will create a new, emtpy file. 

<div style="margin: 2em;"></div>

2. Add the line `options vfio-pci ids=X` to the config file, where `X` is a comma separated list of all the PCI IDs of the devices you want to pass through. In my case `X` is `1002:73ef,1002:ab28`. This specifies which devices the VFI driver needs to be run on. As mentioned in the previous step, this line may be the only line in the file.

<div style="margin: 2em;"></div>

3. Consider adding the line `blacklist snd_hda_intel` to the file. `snd_hda_intel` is a Linux kernel driver responsible for managing High Definition Audio hardware chips for **both Intel and AMD systems**. Presumably this line is intended to improve the guest system's access to audio hardware. Most guides I referenced included this instruction so it seems best to include it. I tried GPU passthrough both with and without this line and did not find a different, though it should be noted I did not rigorously test the audio of the guest or host system. If you have audio issues consider trying the opposite configuration.

<div style="margin: 2em;"></div>

4. In this step, we tell the regular GPU drivers to not run until after the VFIO drivers have started.
 
    - If the GPU you wish to pass through is an AMD GPU, add the lines `softdep amdgpu pre: vfio-pci` and `softdep radeon pre: vfio-pci` to the config file.

    - If the GPU you wish to pass through is an nVidia GPU with proprietary drivers, add the line `softdep nvidia pre: vfio-pci`.

    - If the GPU you wish to pass through is an nVidia GPU with the open source drivers, you should probably add the line `softdep nouveau pre: vfio-pci`. I say "probably" because I did not find any guides suggesting this step, but adding the equivalent lines for my AMD GPU was critical, and `softdep nouveau pre: vfio-pci` follows the pattern of what is needed in the other configurations. 

    <div style="border: 0.17rem solid #000000; padding: 0.7rem; margin: 0.7rem 0rem; border-radius: 0.35rem;">
    
    <details>
        <summary>Tip for testing different configurations.</summary>
        
    <div style="margin: 0.7rem;"></div>
    
    If you aren't sure which of the above lines you need and want to test, you can comment out a line by    adding a `#` at the begginning. For example:
    
    <pre style="margin: 0px"><code> # not sure which of these two lines I need yet...
    # softdep nvidia pre: vfio-pci
    softdep nouveau pre: vfio-pci</code></pre>
    
    </details>
    </div>

<div style="margin: 2em;"></div>

5. Save and exit the file (<kbd>Ctrl + X</kbd> to exit and <kbd>Y</kbd> to save).

<div style="margin: 2em;"></div>

6. Execute `sudo update-initramfs -u` to update the "initial RAM file system". The `initramfs` is a data structure that is loaded into RAM while the computer is booting. It contains the minimal set of kernel modules and device drivers that are requried to boot the system. The `-u` option tells the utility to overwrite the existing file system rather than creating a new one. Running this command after modifying `/etc/modprobe.d/vfio.conf` updates the `initramfs` artifact such that on future boots, our GPU will be loaded with the VFIO drivers and will be accessible to our guest machine. This command can take a while to run (~one minute on my system), wait for it to complete.

<div style="margin: 2em;"></div>

7. Reboot your system.

    **Important Note:** When you reboot your system, any monitors connected to the GPU you are passing through should not output anything (at least once you reach Ubuntu, in my experience they may still show the BIOS). This is intentional and likely means that the VFIO drivers are operating correctly. If the host system can output to these monitors, the guest system will very likely experience an error when it tries to access the GPU. If the guest operating system is Windows, you are likely to get the error "Windows has stopped this device because it has reported problems. (Code 43)".

<div style="margin: 2em;"></div>

8. Execute `lspci -nnk` to view all the PCI devices in your system and the drivers they are using. For the device(s) you are passing through, the `Kernel driver in use:` section should say `vfio-pci`. If it instead says something like `amdgpu`, `nvidia`, or `nouveau`, then something is wrong and the VFIO drivers did not properly take control of the device(s). If this occurs, double check you followed the above instruction correctly. Beyond that I have little advice; good luck.

    <div style="border: 0.17rem solid #000000; padding: 0.7rem; margin: 0.7rem 0rem; border-radius: 0.35rem;">

    <details>
    <summary>The output for the relevant devices on my system looks like this:</summary>
        
    <div style="margin: 0.7rem;"></div>

    <pre style="margin: 0px"><code> 03:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Device [1002:73ef] (rev c1)
	    Subsystem: Gigabyte Technology Co., Ltd Device [1458:2405]
	    Kernel driver in use: vfio-pci
        Kernel modules: amdgpu

    03:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21 HDMI Audio [Radeon RX 6800/6800 XT / 6900 XT] [1002:ab28]
	    Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21 HDMI Audio [Radeon RX 6800/6800 XT / 6900 XT] [1002:ab28]
	    Kernel driver in use: vfio-pci
	    Kernel modules: snd_hda_intel</code></pre>

    </details>
    </div>

&nbsp;<br />
&nbsp;

# 7. Setup Your Virtual Machine Manager

1. Execute `sudo apt-get install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils` to intall:
    - [QEMU](https://www.qemu.org/): A hardware emulator for the CPU. 
    - [KVM](https://www.linux-kvm.org/page/Main_Page) (Kernel-based Virtualization Machine): A virtualization technology that allows the linux kernel to act as a hypervisor. 
    - [livirt](https://libvirt.org/): A virtual machine management tool.
    - bridge-utils: A utilities for configuring ethernet bridges on Ubuntu.
 
<div style="margin: 2em;"></div>

2. Add a user to libvirt by executing `sudo adduser YourUserNameHere libvirtd`. If you get an error telling you that the group "libfirtd" does not exist execute `sudo addgroup libvirtd` then execude the above command again.

<div style="margin: 2em;"></div>

3. Install Virtual Machine Manager by executing `sudo apt-get install virt-manager`. Virtual Machine Manager is a GUI tool for managing and configuring virtual machines.

<div style="margin: 2em;"></div>

4. Launch Virtual Machine Manager by executing `virt-manager` or by pressing the `Super` key, typing `Virtual Machine Manager`, and pressing `Enter`.

<div style="margin: 2em;"></div>

5. In the menu bar go to `Edit > General` and check the `Enable XML editing` box. This makes our life easier later.

&nbsp;<br />
&nbsp;

# 8. Setup Your Virtual Machine

There are multiple ways of setting up a virtual machine. In this guide we will show one way of creating a Windows 10 VM and will apply a few tweaks that will aledgedly improve performance.

*Note to self: I need to go back through these steps and explain why.*

<div style="margin: 2em;"></div>

1. Download a Windows 10 iso file. This can be downloaded on the [Microsoft Website](https://www.microsoft.com/en-ca/software-download/windows10ISO).

<div style="margin: 2em;"></div>

2. Download the virtio drivers for Windows. To do this:
    - Go to [this page](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/?C=M;O=D) of [fedorapeople.org](https://fedorapeople.org).
    - Enter the folder of the most recent release (the top one). At the time of writing this is [virtio-win-0.1.285-1/](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.285-1/).
    - Download the `virtio-win.iso` file by clicking on the corresponding link.

<div style="margin: 2em;"></div>

3. In Virtual Machine Manager, in the top left just under `File` press the `Create a new virtual machine` button.

<div style="margin: 2em;"></div>

4. Choose `Local install media (ISO image or CDROM)` and press the `Forward` button.

<div style="margin: 2em;"></div>

5. Press the `Browse` button, then `Browse Local` in the pop-up windows, navigate to the `.iso` file you downloaded, select it, and press the `Open` button in the top right. Ensure the `Automatically detect from the installation media / source` box is checked and then press `Forward`.
    If you are present with an error saying "You must select an OS." press `OK`, unckeck the `Automatically detect from the installation media / source` checkbox, enter "win10" in the `Choose the operating system you are isntalling:` bar, select `Microsoft WIndows 10 (win10)`, and press the `Forward` button.

<div style="margin: 2em;"></div>

6. Select the amount of memory and CPU cores you want to dedicate to the virtual machine. The optimal configuration of this will depend on what you intend on using your host and guest machines for and the specifications of your hardware. I intend on using my virtual machine for high performance applications such as 3D CAD and so I will dedicate 49152 of my 64018 MB of memory and 12 of my 24 CPU cores to the virtual machine. When you are done specifying the memory and CPU cores press the `Forward` button.

<div style="margin: 2em;"></div>

7. Specify the amount of storage you want to allocate to the virtual machine. Windows 10 requires a minimum of 32 GB of storage to be installed but more is generally required if you want to install sizable applications on it. Once you are done press the `Forward` button.

<div style="margin: 2em;"></div>

8. Check the `Customize configuration before install` checkbox. On this scree you can also specify the name of the virtual machine. Feel free to do this now, however the name, title, and description can all be changed later. Once you are done press the `Finish` button.

<div style="margin: 2em;"></div>

9. In the window that poppped up at the completion of the last step, go to the `Overview` tab and under `Hypervisor Details > Firmware` and select `UEFI x86_64: /usr/share/OVMF/OVMF_CODE_4M.ms.fd`.

<div style="margin: 2em;"></div>

10. Click the `Add Hardware` button in the bottom left. Select the `PCI Host Device` category in the left sidebar of the window that popped up, and select the GPU that you passed through. Complete this for all devices you wish to pass through. For me I have to add two devices, `0000:03:00:0 Advanced Micro Devices, Inc. [AMD/ATI]` and `0000:03:00:1 Advanced Micro Devices, Inc. [AMD/ATI] Navi 21 HDMI Audio [Radeon RX 6800/6800 XT / 6900 XT]`.

<div style="margin: 2em;"></div>

11. Video QXL > Model: QXL (this is the default)

<div style="margin: 2em;"></div>

12. Display Spice > Listen Type: Address, Address: Hypervisor Default, Port: Auto

<div style="margin: 2em;"></div>

12. Boot Options > Check SATA CDROM1, move it to the top

<div style="margin: 2em;"></div>

13. Add Hardware > Storage > Device Type: CDROM device

<div style="margin: 2em;"></div>

14. SATA CDROM2 > Browse > Browse Local > virtio-win.iso

<div style="margin: 2em;"></div>

15. SATA Disk 1 > Disk bus > VirtIO

<div style="margin: 2em;"></div>

16. NIC > Device Model > virtio

<div style="margin: 2em;"></div>

17. Add the following lines:

    ```
    <vendor_id state='on' value='1234567890ab'/>
    <kvm>
     <hidden state='on'/> <!-- idk why this is only one space -->
    </kvm>
    <ioapic driver='kvm'>
    ```

    The end result should look something like:

    ```
    <features>
      <acpi/> <!-- idk if this is right... -->
      <apic>
      <hyperv>
        <relaxed state='on'/>
        <vapic state='on'/>
        <spinlocks state='on' retries='8191'/>
        <vendor_id state='on' value='1234567890ab'/>
      </hyperv>
      <kvm>
        <hidden state='on'/>
      </kvm>
      <vmport state='off'/>
      <ioapic driver=''kvm/>
    </features>
    ```


<div style="margin: 2em;"></div>

18. Click `Begin Installation` in the top left.

<div style="margin: 2em;"></div>

19. Click through Windows install until, "We couldn't find any drivers. To get a storage driver, click Load driver", click `Load driver`, click `OK`, select `Red Hat VirtIO SCSI controller (E:\amd64\w10\viostor.inf)`. Windows should now be able to see the disk. Follow through the install then it should reboot into windows installer.

<div style="margin: 2em;"></div>

20. In Windows go to `CD Drive (E:) virtio-win-[version number]`. Run `virtio-win-guest-tools.exe`.

<div style="margin: 2em;"></div>

21. In Windows, install display drivers for your graphics card.

<div style="margin: 2em;"></div>

22. Reboot your machine and see if it now displays on the monitor connected to your passed through GPU.

<div style="margin: 2em;"></div>

23. Remove the virtual monitor from the VM... need to remember how to do this.

<div style="margin: 2em;"></div>

I am going to get back to performance stuff later so just leaving this here for now.
9. This step is a performance optimization suggested at [this timestamp](https://youtu.be/wxxP39cNJOs?t=531) of a Youtube video. I have no idea what it does so partake in this step at your own risk.
    In the window that pops up after completing the last step 
    Locate the portion
    ```
    <clock offset="localtime">
      <timer name="rtc" tickpolicy="catchup"/>
      <timer name="pit" tickpolicy="delay"/>
      <timer name="hpet" present="no"/>
      <timer name="hypervclock" present="yes"/>
    </clock>
    ```

<div style="margin: 2em;"></div>

Here are links to instructions for two useful things you may want to do. I intend to rewrite these instructions myself when I get the chance. I also intend on adding instructions for passing through a storage drive.
- [sharing folder between linux host and windows guest](https://www.debugpoint.com/kvm-share-folder-windows-guest/)<br />
- [sharing folder between linux host and linux guest](https://www.debugpoint.com/share-folder-virt-manager/)

&nbsp;<br />
&nbsp;

# 9. Trouble Shooting

If something goes wrong, I'm sorry. I don't have a lot of trouble shooting tips. My best advice is to check out the [trouble shooting section](https://askubuntu.com/questions/1406888/ubuntu-22-04-gpu-passthrough-qemu#:~:text=on%20another%20partition.-,TROUBLESHOOTING,-Problem%3A%20The) in this guide's [main source](https://askubuntu.com/a/1410487/1692619).