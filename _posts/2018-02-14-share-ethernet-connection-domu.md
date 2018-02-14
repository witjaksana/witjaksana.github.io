---
layout: post
title: Share ethernet connection to DomU with PV driver
tags: xilinx linux zynq ultrascale+ soc xen
eye_catch: #http://jekyllrb.com/img/logo-2x.png
---

Kill the dhcp client for the ethernet interface
```shell
$ killall -9 udhcpc
```
List and remove existing address from ethernet (eth0)
```shell
$ ip addr show dev eth0
$ ip addr del <your_ip_addr>/24 dev eth0
```
Create the bridge and start dhcp from Dom0
```shell
brctl addbr xenbr0
brctl addif xenbr0 eth0
/sbin/udhcpc -i xenbr0 -b
```
Add xenbridge interface to the configuration file of DomU
```shell
name = "petalinux-guest"
kernel = "/boot/Peta-Kernel"
extra = "console=hvc0 earlyprintk=xenboot root=/dev/xvda1 rw"
memory = 256
vcpus = 1
vif = [ 'bridge=xenbr0' ]
disk = [ 'file:/boot/petalinux-fs.img, xvda, w' ]
```
