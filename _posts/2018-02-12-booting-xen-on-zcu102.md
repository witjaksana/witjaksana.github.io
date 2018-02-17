---
layout: post
title: Booting Xen Hypervisor with Petalinux on Zynq Ultrascale+
tags: xilinx linux zynq ultrascale+ soc xen
eye_catch: #http://jekyllrb.com/img/logo-2x.png
---

The new Zynq Ultrascale+ ZCU102 board is one of the most sophisticated Xilinx board
in the market today. It has four cores of ARM 64-bit, an FPGA, a real-time processing unit,
and a GPU. Like the previous Zynq-7000 platforms, the platform allows to run Linux OS
on the ARM processor, such as Petalinux distribution from Xilinx. Besides that,
the new Petalinux version supports [Xen hypervisor](https://www.xenproject.org/developers/teams/hypervisor.html). 
As we know, many cloud servers provide virtual machine environment based on Xen, which makes
the ZCU102 board a powerful server-class SoC platform.

In Xen-based system, basically there is an OS which is granted the privilege as the administrator (Dom0) and
the guest OSes (DomUs). In this post, I will show the steps on how to boot the Xen hypervisor on yur ZCU102 board
using the Petalinux project. Most of the steps here are based on the [Xilinx wiki page](http://www.wiki.xilinx.com/Building+the+Xen+Hypervisor+with+PetaLinux+2017.3)
with some modifications from me.

### Build and Boot Dom0
Before we can boot the guest OSes on Xen, we need to boot the Xen and Dom0 first. I assume
that you have installed Petalinux on your machine and downloaded the board support package (BSP)
from [Xilinx official download page](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-design-tools.html).

Create the Petalinux project from ZCU102 bsp
```shell
$ petalinux-create -t project -s <BSP_file> -n <Project_name>
```
Go to the main folder and configure the rootfs to enable xen package support
```shell
$ cd <Project_name>
$ petalinux-config -c rootfs
```
From the linux config, enable the package
```shell
Filesystem Packages ---> misc ---> packagegroup-petalinux-xen ---> [*] packagegroup-petalinux-xen
```
Save the configuration and exit.
<!--more-->
We are going to make use a second partition to contain the rootfs. Configure rootfs to initrd
```shell
$ petalinux-config
Image Packaging Configuration ---> Root filesystem type (INITRAMFS) ---> (X) INITRD
```
We will also use a separate device tree file. Change the dtb setting from the config
```shell
-*- Subsystem AUTO Hardware Settings ---> [*] Advanced bootable images storage Settings ---> dtb image settings ---> image storage media (from boot image) ---> (X) primary sd
```
Save the configuration and exit.
We need to modify the xen overlay device tree inside the petalinux project folder. Open
*project-spec/meta-user/recipes-bsp/device-tree/files/xen-overlay.dtsi* and modify the content of the file to become like this
```shell
chosen {
     xen,xen-bootargs = "console=dtuart dtuart=serial0 dom0_mem=768M bootscrub=0 maxcpus=1 timer_slop=0";
     xen,dom0-bootargs = "console=hvc0 earlycon=xenboot root=/dev/mmcblk0p2 rw clk_ignore_unused";

     dom0 {
              compatible = "xen,linux-zimage", "xen,multiboot-module";
              reg = <0x0 0x80000 0x3100000>;
     };
};
```
Modify also the *system-user.dtsi* to become like this
```shell
/include/ "system-conf.dtsi"
/include/ "xen-overlay.dtsi"
/ {
};
```
Anothe modification should also be done in *project-spec/meta-user/recipes-bsp/device-tree/device-tree-generation_%.bbappend*
```shell
SRC_URI_append ="\
    file://system-user.dtsi \
    file://xen-overlay.dtsi \
"
FILESEXTRAPATHS_prepend := "${THISDIR}/files:"
```
If you have a hardware design that you want to put on the FPGA, you can import the hardware description to your petalinux project.
If not, you can skip this process and the petalinux will put the default project.
```shell
$ petalinux-config --get-hw-description=<Vivado_SDK>.sdk
```
Now build the petalinux project
```shell
$ petalinux-build
```
After build finished, package the FSBL, u-boot, and the FPGA bitstream in BOOT.BIN
```shell
petalinux-package --boot --fsbl images/linux/zynqmp_fsbl.elf --fpga images/linux/design_wrapper.bit --u-boot --kernel --force
```
For the second partition, we will use the rootfs generated from petalinux. Go to *images/linux* folder, copy the rootfs file and 
extract it
```shell
$ mkdir -p rootfs
$ cp ./rootfs.cpio ./rootfs
$ cpio -idv < ./rootfs.cpio
```
Copy the BOOT.BIN, Image, and xen.ub files from *images/linux* to the first partition on your SD card.
Now put the SD card and turn on the board. Stop the booting process in u-boot and change the bootcmd value or just execute the command below
```shell
mmc dev $sdbootdev && mmcinfo; load mmc $sdbootdev:$partid 4000000 system.dtb && load mmc $sdbootdev:$partid 0x80000 Image; fdt addr 4000000; load mmc $sdbootdev:$partid 6000000 xen.ub; bootm 6000000 - 4000000
```
You should now running a petalinux as the Dom0.

### Build and Boot DomU
For the DomU, I will also use petalinux. I assume that your ZCU102 board has an internet connection.
Copy the kernel file to the board via ssh transfer
```shell
$ scp images/linux/Image root@zcu102:/boot
```
On your host machine, create a blank 1 Gb to be used as the guest's fs
```shell
$ sudo dd if=/dev/zero of=./domu.img bs=1M count=1024
```
Create a partition table of the image
```shell
$ sudo sh -c 'echo -e "n\np\n1\n\n\nt\n83\nw\n" | fdisk ./domu.img'
```
Make a temporary mounting directory for the guest's fs
```shell
$ mkdir -p ./domu_fs
$ sudo losetup -o 1048576 /dev/loop1 ./domu.img
$ sudo mkfs.ext2 /dev/loop1
$ sudo mount /dev/loop1 ./domu_fs
```
Now unpack the same rootfs for Dom0 in the guest's fs
```shell
$ cp images/linux/rootfs.cpio ./domu_fs
$ cd domu_fs
$ cpio -idv < ./rootfs.cpio
$ rm rootfs.cpio
```
Unmount and detach the fs
```shell
$ sudo umount /dev/loop1
$ sudo losetup -d /dev/loop1
```
Create a guest configuration file in Dom0 petalinux-guest.cfg which contains as follows
```shell
name = "guest0"
kernel = "/boot/Image"
extra = "console=hvc0 earlyprintk=xenboot root=/dev/xvda1 rw"
memory = 128
vcpus = 1
disk = [ 'disk:/boot/domu.img,xvda,w' ]
cpus = [0]
```
Boot the guest
```shell
xl create -c /etc/xen/example-simple.cfg
```
Congratulations, now you are booting another petalinux as a guest OS (DomU).

See also [Share Network Connection to DomU]()
