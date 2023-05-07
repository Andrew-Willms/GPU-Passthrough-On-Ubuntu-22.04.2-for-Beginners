# GPU Passthrough On Ubuntu 22.04.2 for Beginners

**GPU passthrough with an Intel CPU, AMD GPU, and Asus Motherboard on Ubuntu 22.04.2 LTS (including instructions for other hardware).**

_This guide was created on May 3, 2023 and was last updated on May 7, 2023. In case I forget to change the "last updated" date check the commit history._

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
| 9 | [Trouble Shooting](#9-trouble-shooting) |

&nbsp;<br />
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

&nbsp;


### Experience Level

I installed linux for the first time two days before writing these instructions. GPU passthrough can be hard but, hardware gods permitting, you too can achieve high performance graphics in your virtual environment.

In general this guide will be targeted at those who are new to [GNU + Linux](https://www.reddit.com/r/copypasta/comments/63oudw/gnu_linux/).

When I was getting GPU passthrough setup, I found the guides that also explained the purpose of each step invaluable in troubleshooting so I will attempt to do the same. Since I am quite new to this my explanations may not be entirely correct or may be lacking; if you have a better explanation for something and would like to help improve this guide please let me know.

&nbsp;

### General Hardware Requirements

To perform GPU passthrough you must have a CPU, motherboard, and Bios that support IOMMU virtualization (see [Bios Settings](#3-bios-settings) for details). You must also have two GPUs, one of these can be the integrated graphics found on many CPUs. One GPU will display the graphics for the guest system, while the other (this is the one that can be an iGPU) will display the graphics for the host system. It is also very helpful to have at least two monitors, with at least one to connect to each GPU. If you do not have multiple monitors you will have to switch your single monitor between your GPUs and this may be very annoying.

If you only have one GPU GPU passthrough may still be possible; you should look into "Single GPU Passthrough".

My system specifications:
- Intel i9-12900k
- Asus Prime Z690-P Wifi D4 (BIOS version 2212, release date 2022/12/13)
- 64 GB of DDR4 4000 CL18
- Gigabyte Radeon RX 6650 XT
- Ubuntu 22.04.2 LTS

This guide contains the steps I used to enable GPU passthrough on my hardware, however I have attempted to include alternative nVidia and Intel instructions where applicable.

&nbsp;<br />
&nbsp;<br />
&nbsp;

# 3. Bios Settings

[IOMMU](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit) is the technology that will allow our virtual machine to connect directly to the GPU.

The exact settings required to enable IOMMU may vary depending on the platform of your system and the vendor of your motherboard. For Intel based systems enable VT-x and VT-D. For AMD based systems enable SVM. Many BIOSs have I search feature that may help find the appropriate settings.

In my BIOS (Asus Prime Z690-P Wifi D4, BIOS version 2212) I had to enable:
- `Advanced Mode > Advanced \ CPU Configuration \ Intel (VMX) Virtualizatoin Technology`
- `Advanced Mode > Advanced \ System Agent (SA) Configuration \ VT-d: Enabled`
- `Advanced Mode > Advanced \ System Agent (SA) Configuration \ Control Iommu Pre-boot Behavior: Enagle IOMMU during boot`
- `Advanced Mode > Advanced \ System Agent (SA) Configuration \ Graphics Configuration \ Primarry Display: CPU Graphics`

I also noted that `Advanced Mode > Advanced \ CPU Configuration \ Intel VT-x Technology` said `Supported`.

&nbsp;

**Important Note:** In order to get my GPU working with my virtual machine, I had to turn resizable bar ("ReSize BAR" in the top bar of my BIOS) off. I don't knot if this step is required for all systems with support for resizable bar, but it was for mine.

&nbsp;<br />
&nbsp;<br />
&nbsp;

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
    
    represent my GPU and my GPU's HDMI audio interface, respectively (both of which I, and probably you as well, want to pass to your VM). The IDs for these devices on my system are `1002:73ef` and `1002:ab28`.<br />

    &nbsp;

    Note: According to [this timestamp](https://youtu.be/jc3PjDX-CGs?t=220) of one of the Youtube videos linked above, if you wish to pass an IOMMU device to your VM you must pass through all the devices in the same IOMMU group. This can pose challenges if the device you wish to passthrough is not alone in its group (though it seems like GPUs are generally in their own group). Should you run into the a related problem, the linked video persents a work around.

    Note: According to [this part](https://askubuntu.com/questions/1406888/ubuntu-22-04-gpu-passthrough-qemu#:~:text=If%20You%20see%20text%20%22Kernel%20driver%20in%20use%3A%20nvidia%22%20like%20below%3A) of the main source of this guide, the proprietary nVidia drivers don't work with GPU passthrough and will have to be replaced by the Nouveau open source drivers. The link above provides some instruction for this. That being said, in other guides (for example [this one](https://www.youtube.com/watch?v=KVDUs019IB8)) the proprietary nVidia drivers are used.

    &nbsp;

    Other useful commands include:
    - `lscpi` which lists the PCI devices in your system.
    - `lspci -n` which lists the IDs of the PCI devices in your system. (Okay I admit, this one doesn't seem that useful but I wanted to list it for completionism).
    - `lcpci -nn` which lists the PCI devices in your system _and their PCI IDs_.
    - `lscpi -nnk` which lists the PCI devices in your system, their PCI IDs, and some other information about them _including which driver they are using_ (which will be useful later).


&nbsp;<br />
&nbsp;<br />
&nbsp;

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

&nbsp;<br />
&nbsp;<br />
&nbsp;

# 6. Configure VFIO
In thi s section we configure the VFIO drivers. The VFIO drivers run on the host machine instead of the regular drivers and allow the guest machine to access the passed-through GPU.

&nbsp;

1. Edit the configuration file for [VFIO](https://docs.kernel.org/driver-api/vfio.html#:~:text=The%20VFIO%20driver%20is%20an,non%2Dprivileged%2C%20userspace%20drivers.) by executing `sudo nano /etc/modprobe.d/vfio.conf` (or use your text editor of choice). Most likely this configuration file does not yet exist and so running this command will create a new, emtpy file.

&nbsp;

2. Add the line `options vfio-pci ids=X` to the config file, where `X` is a comma separated list of all the PCI IDs of the devices you want to pass through. In my case `X` is `1002:73ef,1002:ab28`. This specifies which devices the VFI) driver needs to be run on.

&nbsp;

3. If your CPU is an Intel CPU add the line `blacklist snd_hda_intel`. I don't know what this line does but several guides included it and it doesn't seem to cause problems on my system. My assumption is that it tells something not to run on startup, but I don't know what that something is or why a CPU driver would be relevant to GPU passthrough.

&nbsp;

4. In this step we tell the regular GPU drivers to not run until after the VFIO drivers have started.
 
    - If the GPU you wish to pass through is an AMD gpu add the lines `softdep amdgpu pre: vfio-cpi` and `softdep radon pre: vfio-pci` to the config file.

    - If the GPU you wish to pass through is an nVidia GPU with proprietary drivers add the line `softdep nvidia pre: vfio-cpi`.

    - If the GPU you wish to pass through is an nVidia GPU with the open source drivers you should probably add the line `softdep nouveau pre: vfio-cpi`. I say "probably" because I did not see any guides suggesting this step but adding the equivalent lines for my AMD GPU was critical and `softdep nouveau pre: vfio-cpi` follows the pattern of what is needed in the other configurations.
If you aren't sure which of these lines you need and want to test, you can comment out lines by adding a `#` at the begginning

    <details>
	<summary>Tip</summary>
	    If you aren't sure which of these lines you need and want to test, you can comment out lines by adding a `#` at the begginning.
    </details>

&nbsp;

5. Save and exit the file.

&nbsp;

6. Execute `sudo update-initramfs -u`. This command can take a while to run (~one minute on my system), wait for it to complete.

&nbsp;

7. Reboot your system.

    **Important Note:** When you reboot your system the monitors connected to the GPU you are passing through should not output anything (at least once you reach Ubuntu, they may still show the BIOS). This is intentional and likely means that the VFIO drivers are operating correctly. If the host operating system is outputting to these monitors you will very likely run into error 43 "Windows has stopped this device because it has reported problems. (Code 43)" on the guest operating system.

&nbsp;

8. Execute `lspci -nnk` to view all the PCI devices in your system and the drivers they are using. For the device(s) you are passing through the `Kernel driver in use:` section should say `vfio-pci`. If it instead says something like `amdgpu`, `nvidia`, or `nouveau` then something is wrong and the VFIO drivers did not properly take control of these devices. My best advice to you is read over the above steps carefully and ensure you executed them correclty.

    <details>
        <summary>The output for the relevant devices on my system looks like this:</summary>
        
        03:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Device [1002:73ef] (rev c1)
	        Subsystem: Gigabyte Technology Co., Ltd Device [1458:2405]
	        Kernel driver in use: vfio-pci
    	    Kernel modules: amdgpu
        03:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21 HDMI Audio [Radeon RX 6800/6800 XT / 6900 XT] [1002:ab28]
	        Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21 HDMI Audio [Radeon RX 6800/6800 XT / 6900 XT] [1002:ab28]
	        Kernel driver in use: vfio-pci
	        Kernel modules: snd_hda_intel
        
    </details>

&nbsp;<br />
&nbsp;<br />
&nbsp;

# 7. Setup Virtual Machine Manager

1. Intall:
    - [QEMU](https://www.qemu.org/): A hardware emulator for the CPU. 
    - [KVM](https://www.linux-kvm.org/page/Main_Page) (Kernel-based Virtualization Machine): A virtualization technology that allows the linux kernel to act as a hypervisor. 
    - [livirt](https://libvirt.org/): A virtual machine management tool.
    - bridge-utils: A utilities for configuring ethernet bridges on Ubuntu.
    
    by executing `sudo apt-get install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils`.

&nbsp;

2. Add a user to libvirt by executing `sudo adduser YourUserNameHere libvirtd`. I don't know if the username you provide needs to be the same as that of your account on the computer but I would assume so.

    If you get an error telling you that the group libfirtd does not exist execute `sudo addgroup libvirtd` then execude the above command again.

&nbsp;

3. Install Virtual Machine Manager by executing `sudo apt-get install virt-manager`. Virtual Machine Manager is a GUI tool for managing and configuring virtual machines.

&nbsp;

4. Launch Virtual Machine Manager by executing `virt-manager` or by pressing the `Super` key, typing `Virtual Machine Manager`, and pressing `Enter`.

&nbsp;

5. In the menu bar go to `Edit > General` and check the `Enable XML editing` box.

&nbsp;<br />
&nbsp;<br />
&nbsp;

# 8. Setup Your Virtual Machine

There are multiple ways of setting up a virtual machine. In this guide we will create a Windows 10 virtual machine with a few tweaks that will aledgedly improve performance.

&nbsp;

1. Download a windows 10 iso file. This can be downloaded on the [Microsoft Website](https://www.microsoft.com/en-ca/software-download/windows10ISO).

&nbsp;

2. Download the virtio drivers for Windows. To do this:
    - Go to [this page](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/?C=M;O=D) of [fedorapeople.org](https://fedorapeople.org).
    - Enter the folder of the most recent release (the top one). At the time of writing this is [virtio-win-0.1.229-1/](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.229-1/).
    - Download the `virtio-win.iso` file by clicking on the corresponding link.

&nbsp;

3. In Virtual Machine Manager, in the top left just under `File` press the `Create a new virtual machine` button.

&nbsp;

4. Choose `Local install media (ISO image or CDROM)` and press the `Forward` button.

&nbsp;

5. Press the `Browse` button, then `Browse Local` in the pop-up windows, navigate to the `.iso` file you downloaded, select it, and press the `Open` button in the top right. Ensure the `Automatically detect from the installation media / source` box is checked and then press `Forward`.
    If you are present with an error saying "You must select an OS." press `OK`, unckeck the `Automatically detect from the installation media / source` checkbox, enter "win10" in the `Choose the operating system you are isntalling:` bar, select `Microsoft WIndows 10 (win10)`, and press the `Forward` button.

&nbsp;

6. Select the amount of memory and CPU cores you want to dedicate to the virtual machine. The optimal configuration of this will depend on what you intend on using your host and guest machines for and the specifications of your hardware. I intend on using my virtual machine for high performance applications such as 3D CAD and so I will dedicate 49152 of my 64018 MB of memory and 12 of my 24 CPU cores to the virtual machine. When you are done specifying the memory and CPU cores press the `Forward` button.

&nbsp;

7. Specify the amount of storage you want to allocate to the virtual machine. Windows 10 requires a minimum of 32 GB of storage to be installed but more is generally required if you want to install sizable applications on it. Once you are done press the `Forward` button.

&nbsp;

8. Check the `Customize configuration before install` checkbox. On this scree you can alos specify the name of the virtual machine. Feel free to do this now, however the name, title, and description can all be changed later. Once you are done press the `Finish` button.

&nbsp;

9. In the window that poppped up at the completion of the last step go to the `Overview` tab and under `Hypervisor Details > Firmware` select `UEFI x86_64: /usr/share/OVMF/OVMF_CODE_4M.ms.fd`.

&nbsp;

10. Click the `Add Hardware` button in the bottom left. Select the `PCI Host Device` category in the left sidebar of the window that popped up, and select the GPU that you passed through. Complete this for all devices you wish to pass through. For me I have to add two devices, `0000:03:00:0 Advanced Micro Devices, Inc. [AMD/ATI]` and `0000:03:00:1 Advanced Micro Devices, Inc. [AMD/ATI] Navi 21 HDMI Audio [Radeon RX 6800/6800 XT / 6900 XT]`.

&nbsp;

11. Video QXL > Model: QXL (this is the default)

12. Display Spice > Listen Type: Address, Address: Hypervisor Default, Port: Auto

12. Boot Options > Check SATA CDROM1, move it to the top

13. Add Hardware > Storage > Device Type: CDROM device

14. SATA CDROM2 > Browse > Browse Local > virtio-win .iso

15. SATA Disk 1 > Disk bus > VirtIO

16. NIC > Device Model > virtio

17. ![image](https://user-images.githubusercontent.com/38842568/236703187-ee1e6439-277f-4d64-b1a9-588856da1437.png)

18. `Begin Installation` in the top left.

19. Click through Windows install until, "We couldn't find any drivers. To get a storage driver, click Load driver", click `Load driver`, click `OK`, select `Red Hat VirtIO SCSI controller (E:\amd64\w10\viostor.inf)`. Windows should now be able to see the disk. Follow through the install then it should reboot into windows installer.

20. In windows go to `CD Drive (E:) virtio-win-[version number]`. Run `virtio-win-guest-tools.exe`.

21. In windows install display drivers for your graphics card.

22. Reboot your machine and see if it now displays on the monitor connected to your passed through GPU.

23. Remove 


&nbsp;

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


&nbsp;

&nbsp;<br />
&nbsp;<br />
&nbsp;

# 9. Trouble Shooting

See the [trouble shooting section](https://askubuntu.com/questions/1406888/ubuntu-22-04-gpu-passthrough-qemu#:~:text=on%20another%20partition.-,TROUBLESHOOTING,-Problem%3A%20The) in this guide's [main source](https://askubuntu.com/a/1410487/1692619).

&nbsp;
