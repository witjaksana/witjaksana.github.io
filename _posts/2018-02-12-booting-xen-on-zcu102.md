---
layout: post
title: Booting Xen Hypervisor with Petalinux on Zynq Ultrascale+
tags: xilinx linux zynq ultrascale+ soc xen
eye_catch: #http://jekyllrb.com/img/logo-2x.png
---

### Build and Boot Dom0
Create a Petalinux project from bsp
```shell
$ petalinux-create -t project -s <BSP_file> -n <Project_name>
```
Configure the root from inside the project folder
```shell
$ petalinux-config -c rootfs
```
Enable Xen
```shell
Filesystem Packages ---> misc ---> packagegroup-petalinux-xen ---> [*] packagegroup-petalinux-xen
```

<!--more-->

Configure rootfs to initrd
```shell
$ petalinux-config
Image Packaging Configuration ---> Root filesystem type (INITRAMFS) ---> (X) INITRD
```
Change dtb setting to separate file
```shell
-*- Subsystem AUTO Hardware Settings ---> [*] Advanced bootable images storage Settings ---> dtb image settings ---> image storage media (from boot image) ---> (X) primary sd
```
Add xen to devicetree
```shell
project-spec/meta-user/recipes-bsp/device-tree/files/xen-overlay.dtsi
```
Modify the line
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
Include the overlay file to the system-user.dtsi
```shell
/include/ "system-conf.dtsi"
/include/ "xen-overlay.dtsi"
/ {
};
```
The same thing should be done in project-spec/meta-user/recipes-bsp/device-tree/device-tree-generation_%.bbappend
```shell
SRC_URI_append ="\
    file://system-user.dtsi \
    file://xen-overlay.dtsi \
"
FILESEXTRAPATHS_prepend := "${THISDIR}/files:"
```
Import hardware description
```shell
$ petalinux-config --get-hw-description=<Vivado_SDK>.sdk
```
Build the petalinux project
```shell
$ petalinux-build
```
Create BOOT.BIN
```shell
petalinux-package --boot --fsbl images/linux/zynqmp_fsbl.elf --fpga images/linux/design_1_wrapper.bit --u-boot --kernel --force
```
Extract rootfs
```shell
$ cpio -idv < ./rootfs.cpio
```
Change U-boot configuration
```shell
mmc dev $sdbootdev && mmcinfo; load mmc $sdbootdev:$partid 4000000 system.dtb && load mmc $sdbootdev:$partid 0x80000 Image; fdt addr 4000000; load mmc $sdbootdev:$partid 6000000 xen.ub; bootm 6000000 - 4000000
```

### Build and Boot DomU

Copy the file to the board
```shell
$ scp images/linux/Image root@zcu102:/boot
```
Create a blank 1 Gb to be used as the guest's fs
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
Unpack the files for fs
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
Create a configuration file in Dom0
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
