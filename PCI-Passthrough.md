Overview
====
## keyword
iommu(vt-d/amd-vi), qemu, kvm, vfio, ovmf, libvirt
## goal
use benifit of linux (host) and windows (guest, with pci-passthrough) at the same time, switch GPU between host and guest without restart (neither kernel nor X)

Related CLI
====
kernel module
----
* `lsmod | grep nvidia` list all kernel module that about nvidia
* `modprobe vfio-pci` (the same as `modprobe vfio_pci`) load module "vfio_pci" into kernel (could not load 2 modules in one modprobe like `modprobe vfio vfio-pci`, the latter one "vfio-pci" would be considered as the argument of "vfio")
* `modinfo i915` show info of module

qemu
----
* `qemu` often actually means `qemu-system-x86_64` https://wiki.archlinux.org/index.php/QEMU
* `qemu-img create -f qcow disk.img 120G && qemu-img info disk.img` https://www.cnblogs.com/gaott/archive/2012/06/29/2569840.html (cn)
* `qemu --enable-kvm ...` (equals to `--machine accel=kvm`) https://lists.nongnu.org/archive/html/qemu-discuss/2013-01/msg00015.html (the new way)
* qemu options:
  * `-machine pc-i440fx-2.11,accel=kvm,usb=off,vmport=off,dump-guest-core=off`
  * `-cpu host`
  * `-m 4096` memory
  * `-realtime mlock=off`
  * `-smp 4,sockets=1,cores=2,threads=2` cpu topology
  * `-rtc base=utc,driftfix=slew`
  * `-boot strict=on`
  * `-device ...`
  * `-drive ...`

libvirt
---
* `virsh` should be run through `sudo` if you could not see anything in `virsh list --all`
* `virsh dumpxml win10` you could specify it via name or id
* `virsh edit win10` it would use the environment variable `EDITOR=vim`

Tutorial
====
basic steps
----
0. enable "IOMMU" in kernel (see advanced below)
1. remove nvidia_drm module before X start (due to "issue about systemd", see below) `sudo rmmod nvidia_drm`
2. `systemctl start libvirtd` if not enabled (see "libvirt and ovmf" below to setup libvirt)
3. `startx` and open `virt-manager`
4. create new virtual machine
  * Install the operating system via "Local install media (ISO image or CDROM)" and choose the "install-windows.iso" file
  * I suppose to "Select or create custom storage" otherwise it would create in "/var/liblibvirt/images" (Prefer qcow, since it could allocate on use)
  * MUST choose "Costumize configuration before install" before finish in order to use ovmf/uefi
  * change Overview->Firmware to UEFI, change CPU->Model to "host-passthrough" (maybe you have to manually input here if not present)
  * run in command `virsh edit <your_vm_name>` and add "<kvm><hidden state='on'/></kvm>" in section "domain->features" (see "code 43 for nvidia driver on windows" below)
5. install windows 10 as usual (see "windows utc timezone support" below for some setup)
6. (shutdown) view details of your vm and add passthroughs (then start your vm)
  * "Add Hardware"->"PCI Host Device" to add your GPU (and with audio of your GPU)
  * "Add Hardware"->"USB Host Device" to add a usb keyboard (maybe necessary since the simulated one is stupid)
  * "Add Hardware"->"Storage" to add SSD, "Select or create custom storage":"/dev/sdX"(choose your own X here), "Device Type":"Disk Device", "Bus Type":"Sata"(in most cases)
7. if stuck at boot, consider remove "Video" and "Graphics".

advanced
----
https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF  
https://www.waitig.com/libvirt%E4%B9%8Bvfio-pci%E7%9A%84passthrough.html (cn) example  
1. make sure "IOMMU" is enabled
  * `dmesg | grep -e DMAR -e IOMMU` should output
    > ACPI: DMAR 0x... ... (v01 INTEL  SKL      00000001 INTL 00000001)  
    > Intel-IOMMU: enabled
  * add "intel_iommu=on iommu=pt" in mode line of kernel (or "amd_iommu")
2. your GPU should be in dedicated iommu_group
  * ls /sys/kernel/iommu_groups/*/devices/
  * ls -l /sys/bus/pci/devices/0000:01:00.0/
3. unbind GPU https://www.kernel.org/doc/Documentation/vfio.txt
  * `lspci`, `lspci -n`, `lspci -nn`, `lspci -nnk` (find VGA/3D)
  * if the output is "01:00.0 0300: 10de:1b81 (rev a1)"
  * `echo "0000:01:00.0"|sudo tee /sys/bus/pci/devices/0000:01:00.0/driver/unbind`
    "/sys/bus/pci/devices/0000:01:00.0/driver" should be empty
  * `echo "10de 1b81"|sudo tee /sys/bus/pci/drivers/vfio-pci/new_id`
    "/sys/bus/pci/devices/0000:01:00.0/driver" should link to "/sys/bus/pci/drivers/vfio-pci"
  * `echo "0000:01:00.0"|sudo tee /sys/bus/pci/drivers/vfio-pci/bind`
    optional, force bind driver to vfio-pci
  * `echo "10de 1b81"|sudo tee /sys/bus/pci/drivers/vfio-pci/remove_id`
    optional
  * and for `0000:01:00.1` also
4. (after vm exit/shutdown) rescan GPU
  * `echo 1 |sudo tee /sys/bus/pci/devices/0000:01:00.0/remove`
  * `echo 1 |sudo tee /sys/bus/pci/rescan`
5. for bind GPU to vfio-pci at boot, see wiki (modify /etc/modprobe.d/vfio.conf and /etc/mkinitcpio.conf)

FAQ
====
anti-cheat system of steam
----
https://steamcommunity.com/discussions/forum/9/135510393198367927 TL;DR you have to choose between VM and STEAM

windows host doesn't support vt-d
----
https://superuser.com/questions/540905/can-i-use-vt-d-with-a-windows-host-for-a-vm (Oct 2013)  
social.technet.microsoft.com [Single Root I/O Virtualization (SR-IOV)](https://social.technet.microsoft.com/Forums/windows/en-US/fc1eba31-acec-4a1a-9010-b2e06b28082c/hyperv-intel-vtd?forum=winserver8gen) (Jul 2012)  
https://software.intel.com/en-us/blogs/2009/02/24/step-by-step-guide-on-how-to-enable-vt-d-and-perform-direct-device-assignment (2009 VMware) net adapter  

detach gpu on the fly
----
https://www.reddit.com/r/linux/comments/4svj5j/vga_passthrough_with_only_one_gpu/d5d33um/  
https://www.reddit.com/r/VFIO/comments/4owzs2/gpu_passthrough_with_only_one_gpu/  
For nvidia `modprobe -r nvidia_drm nvidia_modeset nvidia_uvm nvidia`

xorg only use integrated graphics card
----
https://gist.github.com/alexlee-gk/76a409f62a53883971a18a11af93241b Configure xorg.conf  
For arch linux, `rm -f /usr/share/X11/xorg.conf.d/10-nvidia-drm-outputclass.conf` if you installed package "nvidia-utils" which prevents xorg(startx) use "nvidia-drm" module.
https://www.x.org/releases/current/doc/man/man5/xorg.conf.5.xhtml  
search locations: /etc/X11/xorg.conf /etc/X11/xorg.conf.d /usr/share/X11/xorg.conf.d  
`pacman -Ql nvidia|grep -i xorg`
> /usr/lib/nvidia/xorg/libglx.so  
/usr/lib/xorg/modules/drivers/nvidia_drv.so  
/usr/share/X11/xorg.conf.d/10-nvidia-drm-outputclass.conf  
```
Section "OutputClass"
    Identifier "intel"
    MatchDriver "i915"
    Driver "modesetting"
EndSection
Section "OutputClass"
    Identifier "nvidia"
    MatchDriver "nvidia-drm"
    Driver "nvidia"
    Option "AllowEmptyInitialConfiguration"
    Option "PrimaryGPU" "yes"
    ModulePath "/usr/lib/nvidia/xorg"
    ModulePath "/usr/lib/xorg/modules"
EndSection
```
https://lists.x.org/archives/xorg-devel/2014-February/040564.html patch for OutputClass  

nvidia/libgl related package in arch linux
----
* libvdpau: Video Decode and Presentation API for Unix https://en.wikipedia.org/wiki/VDPAU
* mesa
* virtualgl
* nvidia-utils: provides vulkan-driver opengl-driver nvidia-libgl (nvidia-smi libglx.so nvidia_drv.so ...)
* nvidia: depends on nvidia-utils and/or libgl
* https://git.archlinux.org/svntogit/packages.git/tree/trunk/PKGBUILD?h=packages/nvidia-utils

kvm vs xen
----
https://www.zhihu.com/question/19844004 (cn) TL;DR just use kvm

vitrual GPU/vgpu
----
https://www.nvidia.com/object/grid-technology.html nvidia grid not support kvm (and it is not free of charge)  
https://feisky.xyz/virtualization/GPU/ only amd support SR-IOV  
https://github.com/Seitanas/kvm-vdi/issues/66 KVM Virtualization and NVIDIA GRID Compatibility  
http://www.tomsitpro.com/articles/red-hat-nvidia-grid-kvm-quadro,1-1411.html (nov 2013) redhat is working on grid&kvm https://blogs.nvidia.com/blog/2013/11/15/visualizing-the-answers/  
http://docs.nvidia.com/grid/5.0/grid-vgpu-release-notes-red-hat-el-kvm/index.html Virtual GPU Software R384 for Red Hat Enterprise Linux with KVM Release Notes (RedHat only?)  
https://forums.geforce.com/default/topic/1020713/nvidia-vgpu-kvm-linux-instruction/ (2018-08) no  
https://www.kraxel.org/blog/tag/vgpu/ (2017-01) intel?  
http://www.brianmadden.com/opinion/NVIDIA-AMD-and-Intel-How-they-do-their-GPU-virtualization Intel (GVT-g), AMD (MxGPU), and NVIDIA (vGPU)  
http://blog.sciencenet.cn/blog-279072-1035996.html KVMGT (intel GVT-g) in linux kernel  
https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions (maybe like special api in *.so that nvidia-docker translate cuda)  

what is /dev/dri/card* (0 or 1)
----
http://www.wowotech.net/linux_kenrel/dri_overview.html (cn)  
glx(.so) is wrap of gl in X (X server way)  
dri is application directly communicate with driver (for rendering but not composition, wayland way)  
DRI -> DRM(direct rendering module) -> gpu -> returning image buffer -> KMS(kernel mode setting)  
(according to /var/log/Xorg.0.log)  
(II) xfree86: Adding drm device (/dev/dri/card0)  
(II) systemd-logind: got fd for /dev/dri/card0 226:0 fd 11 paused 0  
(II) xfree86: Adding drm device (/dev/dri/card1)  
(II) systemd-logind: got fd for /dev/dri/card1 226:1 fd 12 paused 0  

how to unload nvidia driver/issue about systemd
----
https://stackoverflow.com/questions/448999/is-there-a-way-to-figure-out-what-is-using-a-linux-kernel-module/449179#449179  
lsof/fuser helps a lot (lsof cheatsheet http://www.jusene.me/2017/03/31/lsof/), although it's not always useful  
https://wiki.archlinux.org/index.php/kernel_modules  
https://devtalk.nvidia.com/default/topic/1024407/nvidia_drm-remains-in-use-for-no-apparent-reason-after-xorg-shutdown-/ systemd-logind is the issue  
https://github.com/systemd/systemd/issues/6908 DRM devices opened by logind stay referenced indefinitely by PID 1  
https://github.com/systemd/systemd/blob/master/rules/60-drm.rules refer drm (not know how this file works)  
https://github.com/systemd/systemd/blob/4050e4797603d3644707d58edfd9742b5311c7cf/src/login/logind-session-device.c#L188 session_device_start: sd->type==DEVICE_TYPE_DRM => sd_drmsetmaster(sd->fd)  
for (sd: s->devices) // https://github.com/systemd/systemd/blob/3219f05c1d55fa8264cbf136a63e1e10d080cb90/src/login/logind-device.h  

remove nvidia drivers before X starts
----
https://arseniyshestakov.com/2016/03/31/how-to-pass-gpu-to-vm-and-back-without-x-restart/ (from 3)  
for nvidia_drm it works, after X starts, it's not reloaded (10-nvidia.conf removed in /usr/share/X11/xorg.conf.d)  
for nvidia_uvm and nvidia has nothing to do with X (source request?), so we could easily load and unload it (python with mxnet could properly take and release it)  
nvidia-smi would automatically load nvidia (but not other drivers), and mxnet would load nvidia_uvm when training (at least not when import)  
`sudo rmmod nvidia_modeset nvidia_uvm nvidia`

what's VFIO, DMA, libvirt, OVMF
----
[VFIO DMA and user mode](https://ggaaooppeenngg.github.io/zh-CN/2017/06/05/VFIO-%E2%80%94%E2%80%94%E5%B0%86-DMA-%E6%98%A0%E5%B0%84%E6%9A%B4%E9%9C%B2%E7%BB%99%E7%94%A8%E6%88%B7%E6%80%81/) (cn)

libvirt and ovmf
----
bbs.archlinux.org [libvirt via virt-manager virtual network start failed](https://bbs.archlinux.org/viewtopic.php?id=198744)
https://wiki.archlinux.org/index.php/Libvirt#Server  
`pacman -S ebtables dnsmasq` and `usermod -a -G libvirt <username>` and edit "/etc/libvirt/qemu.conf"
```
nvram = [
  "/usr/share/ovmf/x64/OVMF_CODE.fd:/usr/share/ovmf/x64/OVMF_VARS.fd"
	#"/usr/share/ovmf/ovmf_code_x64.bin:/usr/share/ovmf/ovmf_vars_x64.bin"
]
```
then `systemctl start libvirtd`  
virt-manager (gui)  
https://computingforgeeks.com/virsh-commands-cheatsheet/  
https://www.centos.org/docs/5/html/5.2/Virtualization/chap-Virtualization-Managing_guests_with_virsh.html  
add GPU: details -> add hardware -> select PCI device  
add SATA: details -> add hardware -> storage: /dev/sdb disk device/SATA  
restart not take effect to device change (must turn off and start again)  
https://libvirt.org/guide/html/Application_Development_Guide-Device_Config-PCI_Pass.html simple example  
https://www.reddit.com/r/VFIO/comments/645vhm/how_do_i_pass_through_a_hard_disk_drive_so_that_i/  
https://lime-technology.com/forums/topic/46634-passthrough-hard-drive-to-vm/  

seabios vs uefi(ovmf)
----
libvirt did not detect any uefi/ovmf firmware image installed on the host (fix /etc/libvirt/qemu.conf)  
https://lime-technology.com/forums/topic/45877-ovmf-or-seabios-what-is-best-and-why/
just use ovmf  

code 43 for nvidia driver on windows
----
https://stackoverflow.com/questions/41362981/nvidia-gpu-passthrough-fail-with-code-43  
in question, hide the hypervisor by adding "kvm=off" and "hv_vendor_id=123456780ab"  
in answer, put an UEFI-enabled ROM as "romfile=", make sure to triple-check that your card is UEFI-ready, especially on stuff that is older than ~2014 (outdated information)  
```
    <hyperv>
      <vendor_id state='on' value='1234567890ab'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
```
tested kvm is necessary and hyperv is optional?

windows utc timezone support
----
edit registry "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\TimeZoneInformation\RealTimeIsUniversal" as `(DWORD) 1`
