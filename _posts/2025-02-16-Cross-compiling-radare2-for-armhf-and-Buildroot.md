---
title: Cross compiling radare2 for armhf and Buildroot
date: 2025-02-16 15:00:00 +0200
categories:  #[ TOP_CATEGORIE, SUB_CATEGORIE]
tags: []     # TAG names should always be lowercase
---

## Introduction
In this article, we will learn how to compile [radare2](https://github.com/radareorg/radare2) for `armhf` using Buildroot's toolchain (note that this can be adapted to any other cross-toolchain you have, either you got it from your package manager, or with [Crosstool-NG](https://crosstool-ng.github.io)).

_I assume you are familiar with Buildroot and already know how to use it to build a Linux system._

## Steps
First, clone the `radare2` repo:
```bash
$ git clone https://github.com/radareorg/radare2
```

Then, build a toolchain using Buildroot:
```bash
# assuming Buildroot is ready in its dedicated folder
$ cd ~/buildroot
$ make qemu_arm_vexpress_defconfig
$ make menuconfig
$ make toolchain
```

This will give you a full cross-toolchain in `buildroot/output/host/`.\
The sysroot will be in `buildroot/output/host/arm-buildroot-linux-gnueabi*/sysroot`.

Next, create an overlay, i.e. a folder that will receive additional files that Buildroot will copy to its rootfs at the end of the build so that they are available to the target system.
```bash
$ mkdir ~/buildroot/overlay
```

Now, configure radare2 to use Buildroot's cross-compiler, and to output it to our overlay:
```bash
$ cd ~/radare2
$ CC=$HOME/buildroot/output/host/bin/arm-buildroot-linux-gnueabihf-gcc CFLAGS=-I$HOME/output/host/include LDFLAGS=-L$HOME/buildroot/output/host/lib ./configure --prefix=$HOME/buildroot/overlay/usr/local/r2 --target=armhf-buildroot-linux --host=armhf-buildroot-linux
$ make -j 8
$ make install
```

Since we installed r2 to `/usr/local/r2`, its binaries and libs won't be found be default, so we need to edit `LD_LIBRARY_PATH` and `PATH` of our new system.\
First, we can alter the `PATH` by setting the `BR2_SYSTEM_DEFAULT_PATH` to include our r2 folder.\
As for `LD_LIBRARY_PATH`, we can set it by creating `/etc/profile`:
```bash
$ cd ~/buildroot/overlay
$ mkdir etc
$ cat "export LD_LIBRARY_PATH=/usr/local/r2/lib:$LD_LIBRARY_PATH" >> overlay/etc/profile
```

You can now build your rootfs with Buildroot, and do not forget to add the overlay to your build by settings `BR2_ROOTFS_OVERLAY` to `$HOME/buildroot/overlay`.\
Then build:
```bash
$ cd ~/buildroot
$ make menuconfig
# set BR2_ROOTFS_OVERLAY and BR2_SYSTEM_DEFAULT_PATH, save and exit
$ make
```

## Conclusion

Now your system has radare2 installed!
```bash
Welcome to Buildroot
buildroot login:
$ r2
Usage: r2 [-ACdfjLMnNqStuvwzX] [-P patch] [-p prj] [-a arch] [-b bits] [-c cmd]
          [-s addr] [-B baddr] [-m maddr] [-i script] [-e k=v] file|pid|-|--|=
```