# 1. The problem

On Linux, it can be hard to render a desktop environment with Blackwell series eGPUs. The main barricades you will face are as follows:

- NVIDIA 50 series GPUs (Blackwell), as noted on their website (https://developer.nvidia.com/blog/nvidia-transitions-fully-towards-open-source-gpu-kernel-modules/), only support the open-source driver. 
- When using the open-source driver, Linux has to load the GSP firmware from the GPU itself, instead of from the driver (like the proprietary driver does). 
- Loading the GSP from the GPU requires a (relatively) large, continuous block of memory to be assigned to the GPU, and the GPU needs to manage that part of memory directly, which is known as **Direct Memory Access (DMA)**. 
- Thunderbolt devices are not like PCIe devices. PCIe devices are normally trusted by the Linux kernel, so operations like DMA are normally allowed. This allows GPUs connected via PCIe to use features like ReBAR and load the GSP. However, for most motherboards, especially motherboards on laptops, they are set by their manufacturer to ***mistrust*** every device by default until manually authorized by the user. So, if connected via Thunderbolt, the kernel will refuse to allocate memory for the GSP, thus the GPU can never get recognized. Furthermore, Thunderbolt authorization tools like bolt (on Arch) won't work because they load after the kernel. 
- For many laptops, the manufacturer has hidden the corresponding BIOS option, which means it is not possible to set "trust all connected Thunderbolt devices" (https://wiki.archlinux.org/title/Thunderbolt#User_device_authorization) in the BIOS GUI provided by the manufacturer.

# 2. How to solve this

After days of trial and error, I found that the only way to connect a 50 series GPU is to disable **Kernel DMA Protection**. 

If you are using Windows on the same computer, you can check if this is already disabled in System Info on Windows. There should be a row for "Kernel DMA protection", check its status there. 

It would be great if you could set this to disabled directly in your BIOS GUI. If you can, and you are not using dual boot (which means you do not use Windows and Linux on the same computer), you can just disable it and go straight to 2.2. ***However, if you do use dual boot, no matter if Windows and Linux are on the same hard disk or not, please read 2.1, or you might run into problems when booting into Windows.***

## 2.1 Disable Kernel DMA Protection

Although most laptop BIOSes do not have this option in the BIOS GUI, it is still possible to disable it by directly editing the BIOS config at the lowest level (i.e., changing the corresponding bits inside the BIOS flash directly) using tools like **RU.efi**. 

If you are using dual boot, please make sure you have turned off **Memory Integrity** in your Windows settings. You might run into problems when Kernel DMA Protection is disabled while Memory Integrity is on. 

> Note: Some very strict anti-cheat software may refuse to load if Kernel DMA Protection and Memory Integrity are disabled. They enforce this to prevent people from using DMA cheats in their games. If you play these games, for example, CS2 on a non-official platform like Perfect World (not tested), you may choose to either keep this setting enabled and give up your eGPU on Linux (normally, 50 series eGPUs work well on Windows), or try at your own risk (I think normally your account will not get banned because of this; a responsible game company should recover your account if they ban you because of this by mistake). 

The following guide is done on a laptop which has a Windows 11 and Arch dual boot configured. If you only have Linux installed, this guide still works, but you have to find Linux alternatives for some of the software used here. There is theoretically nothing that can only be done on Windows in this guide. 

As I'm writing this on Linux now, there will not be screenshots from Windows, but I'm confident that this guide is detailed enough so that you can follow it easily. 

### 2.1.1 Disable Memory Integrity in Windows

Go to Windows settings, on the left sidebar, click **Privacy and Security**, then click **Device Security**, **Core Isolation**, and you will see **Memory Integrity**; disable it. 

### 2.1.2 Extract your BIOS

Normally, a BIOS file downloaded directly from the manufacturer's website won't work as they are usually encrypted. If you open an encrypted BIOS file using tools like UEFITool (https://github.com/LongSoft/UEFITool), you cannot find the correct `setup` module. However, a BIOS extracted from the BIOS flash should be decrypted and ready to use. 

I dumped my BIOS on Arch (of course, the system boots while not connected to the eGPU) using the following command. You can dump it with your preferred tool. 

```
# Do "pacman -S flashrom" if it is not installed
sudo flashrom -p internal --ifd -i bios -r bios_region.rom 
```

Then use a tool like UEFITool to open the generated rom file, find the setup area, right-click, and export it. 

After that, use ifextractor (https://github.com/LongSoft/IFRExtractor-RS) to extract the exported file into a `txt` file; this `txt` file should contain all the options supported by the BIOS. Open the `txt` file with your preferred editor. 

### 2.1.3 Find proper offset 

Depending on your motherboard manufacturer, the precise name of this BIOS option may vary, but you should find it by searching for "DMA" in this `txt` file. 

For my AMI-based BIOS, it looks like the following: 

```
OneOf Prompt: "DMA Control Guarantee", Help: "Enable/Disable
DMA_CONTROL_GUARANTEE bit", QuestionFlags: 0x10, QuestionId: 0x62B, VarStoreId: 0x5, VarOffset: 0x87, Flags: 0x10, Size: 8, Min: 0x0, Max: 0x1, Step: 0x0

OneOfOption Option: "Enabled" Value: 1, Default, MfgDefault

OneOfOption Option: "Disabled" Value: 0 
```

From the search result above, the proper offset of this setting on ***MY COMPUTER*** is `0x87` (VarOffset), and it is under module `0x5` (VarStoreId). We can see it is enabled by default, and we want to set it to disabled. 

Then we are going to find what module `0x5` actually stands for. Search for `VarStoreId: 0x5`, or just `0x5`, and you should find something near the beginning of this `txt` file like the following:

```
VarStore: VarStoreId: 0x5 [GUID], Size: 0x..., Name: SaSetup
```

 Then we know it is under the module `SaSetup`.
 
 Do this on your computer and find the proper offset for your device. Take note of the offset, as we won't be able to see this `txt` file when tweaking the BIOS bits. 
 
### 2.1.4 Set up RU.efi Download RU.efi

Download RU.efi here: https://github.com/JamesAmiTw/ru-uefi. Note that the recent few versions fail to run (after a flash of a black screen, you get sent back to the BIOS GUI instead of getting into the tool). I tried version 5.34.0426 and it worked. The extraction password can be found in the author's personal blog (the link is on the GitHub page of this tool). 

If you are on Windows, right-click the Windows logo and click Disk Management. Shrink 100 MB of unallocated space from your main volume. Create a volume with that space using FAT32 (you can keep clicking Next to use default settings; when a dropdown labelled NTFS appears, change that to FAT32 and continue to click Next until finished). 

Put the downloaded RU.efi under the directory `D:\EFI\Boot\BOOTX64.efi` (assuming `D:` is the new FAT32 volume, create the parent folders as named above, and rename the efi file to `BOOTX64.efi`). 

Of course, you can use a USB storage device using FAT32 filesystem instead of this FAT32 volume. And you can create this FAT32 volume using Linux if you like. 

### 2.1.5 Change BIOS settings

Reboot and get into the BIOS, and choose to boot from this FAT32 volume (it should show up in your boot options if the steps above were done correctly). You may have to press Enter to close the floating window if it shows up. 

Then press `Alt` and `=` together; you should see all the modules in your BIOS. Go to the module where your DMA protection setting is located (for me, it is `SaSetup`). Press Enter to get into the module. Use the arrow keys to go to the proper offset (the currently selected offset under the cursor is shown in the upper-left corner; move the cursor until it reaches the correct offset). 

After that, you should check that the block you are going to modify is ***exactly the block you are looking for!*** Remember the `txt` file shows: 

```
Min: 0x0, Max: 0x1, Step: 0x0
```

And remember the default is `1`, so if the block value is anything other than `1`, ***DO NOT TOUCH IT!*** Go back to that `txt` file and check what went wrong. You must have gotten into either the wrong module or the wrong offset. 

If you are confident you are on the right track, just type `0` when the cursor is in the intended place (no need to press Enter), then move the cursor to another place to confirm (again, no need to press Enter). Use `Ctrl` + `W` to save, then `Ctrl` + `Alt` + `Delete` to reboot. If your system still boots, then congrats, it is very likely you did it correctly. 

You can check if Kernel DMA protection is disabled in System Info on Windows. 

## 2.2 Change Grub startup arguments

It is tested that the following Grub arguments work: 

```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet mem_sleep_default=deep pcie_ports=native pcie_aspm=off pci=nocrs,assign-busses,realloc,hpbussize=0x33,hpmmiosize=256M,hpmmioprefsize=5G,noaer pci-stub.ids=10de:24a0,10de:228b intel_iommu=off nvidia_drm.modeset=1 nvidia_drm.fbdev=1 nvidia.NVreg_DynamicPowerManagement=0x02" 
```

Among them, these are absolutely necessary: 

```
pcie_ports=native pcie_aspm=off pci=nocrs,assign-busses,realloc,hpbussize=0x33,hpmmiosize=256M,hpmmioprefsize=5G,noaer
```

 As tweaking those args is too tedious (I was tweaking them in tons of different ways for a whole week), I haven't tried whether the eGPU will work without any of the following: 

```
intel_iommu=off nvidia_drm.modeset=1 nvidia_drm.fbdev=1 nvidia.NVreg_DynamicPowerManagement=0x02 
```

In theory, iommu is off by default, and if you are using the latest driver, `nvidia_drm` should be automatically set by nvidia-utils, and I believe `DynamicPowerManagement=0x02` should also be the default. You can try to remove them and test whether it still works if you want to. 

### 2.2.1 Blocking dGPU

If your laptop has a dGPU, perhaps you have to block it, as I tried and found it is impossible to boot while 3 GPUs (integrated Intel GPU, the dGPU that came with the laptop, and the 50 series eGPU) are connected at the same time. The reason is likely due to an address conflict during Linux startup. Actually, they have problems on Windows too; if I don't set it to Integrated Only mode using the software provided by the laptop manufacturer, every motion will lag, and you cannot even play a video on YouTube or use a live wallpaper. 

**Do not use tools like `supergfxctl` to block your dGPU**; the way `supergfxctl` activates Integrated mode is by completely blocking the NVIDIA driver, so you won't be able to use your eGPU that way. The proper way to block the dGPU is to use `pci-stub` in your boot args. 

First, you have to get the ID of your dGPU and its high-definition audio device. Use: 

```
lspci -nnk -d 10de:
```

And you will see the following: 

```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA104 [Geforce RTX 3070 Ti Laptop GPU] [10de:24a0] (rev a1)
	Subsystem: ASUSTeK Computer Inc. Device [1043:1c92]
	Kernel driver in use: pci-stub
	Kernel modules: nouveau, nvidia_drm, nvidia
01:00.1 Audio device [0403]: NVIDIA Corporation GA104 High Definition Audio Controller [10de:228b] (rev a1)
	Subsystem: ASUSTeK Computer Inc. Device [1043:1c92]
	Kernel driver in use: pci-stub
	Kernel modules: snd_hda_intel
```

Then add the IDs in the brackets to your Grub boot args: 

```
pci-stub.ids=10de:24a0,10de:228b
```

After finishing your changes, remake the Grub config to make it work. On Arch, this is done through: `sudo grub-mkconfig -o /boot/grub/grub.cfg` (remember to match the output path). 

## 2.3 Pack Thunderbolt into your startup image

I've done this on Arch; for other distributions, check the corresponding documentation for where the file is. 

Modify `/etc/mkinitcpio.conf`, and change the modules line to the following: 

```
MODULES=(thunderbolt) 
```

Remember that everything related to NVIDIA should load after `thunderbolt`; I loaded these drivers through the kms hook. If you really need early KMS, putting every module related to NVIDIA behind thunderbolt should work. 

Then recreate your image. On Arch, this is done through `sudo mkinitcpio -P`. 

## 2.4 Reboot

Shut down and connect your eGPU, boot up, and everything should work fine. Your eGPU should appear in `nvidia-smi`, and monitors connected to the eGPU should be recognized by the system. 

If your eGPU still disappears in `nvidia-smi`, use `bolt list` to check whether its status is `authorized`. If the status is just `connected`, copy its UUID and do: 

```
bolt forget <uuid>
bolt enroll <uuid>
```

 Then the eGPU should be recognized (perhaps you need to boot again to have every driver loaded properly). 
 
 After that, you can configure your display so that you can use your external monitor. Perhaps you should also set up preserve video memory so that your system will still be able to wake up after suspending. Guides on these can easily be found on websites like the Arch Wiki. 

# 3. A note on what settings do not work

This is a note on settings that won't solve the problem, so you don't have to try these settings again in vain. 

- Disable VT-d in BIOS. This allows `nvidia-smi` to detect your eGPU, but monitors connected to the eGPU will not be detected by your desktop environment (e.g., Hyprland, `hyprctl monitors all` only shows the built-in monitor of the laptop), and any process that uses the eGPU (e.g., torch, diffusion) will result in an instant freeze of the GUI and a kernel panic afterwards. 
- Enabling iommu in the BIOS startup args (i.e., `intel_iommu=on`); this won't work regardless of whether you use `iommu=pt` along with `intel_iommu=on` or not. Both the passthrough mode and normal mode will give an error: 

```
NVRM: GPU0 osIovaMap: osIovaMap: failed to map allocation (status = 0x59)
```

- Assigning `swiotlb=131072`, or assigning CMA (whether on a lower address `cma=512M@0-4G` or on a higher address `cma=512M`). 
- Recommended settings on the Arch Wiki (https://wiki.archlinux.org/title/External_GPU): `pcie_ports=native pci=assign-busses,hpbussize=0x33,realloc,hpmmiosize=128M,hpmmioprefsize=16G`. As `hpmmioprefsize=16G` is usually too large for PCIe x4, this will take up `16G x 4 = 64G` of address space, which will give an error on my device (the precise reason is not clear; my device has 32 GB of RAM). 

Currently, I have not found a possible solution to use the dGPU and eGPU at the same time (i.e., you must block your dGPU). I successfully boot into the system when I use Discrete mode (only dGPU), but it seems impossible to either get into the desktop environment or use any program that requires the eGPU (e.g., train a model); any of these will cause a sudden freeze for an unknown reason (it cannot even be found in the system log).

Feel free to open an **issue** if you run into any undocumented problems or have questions regarding this guide. If you have found better configurations, alternative tools for other Linux distributions, or solutions to use both dGPU and eGPU simultaneously, **pull requests** are more than welcome! Your contributions help make this guide more robust and may help others in need.
