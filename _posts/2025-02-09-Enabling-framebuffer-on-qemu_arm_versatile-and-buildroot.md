---
title: Enabling framebuffer on qemu_arm_versatile and Buildroot
date: 2025-02-09 17:00:00 +0200
categories:  #[ TOP_CATEGORIE, SUB_CATEGORIE]
tags: []     # TAG names should always be lowercase
img_path : /2025-02-09-QEMU-FRAMEBUFFER
---

## Introduction

When using the default Buildroot target `qemu_arm_versatile`, no display is enabled:

![Desktop View](empty.png){: width="300" .shadow } 
_That's sad and disappointing_

In this tutorial, we will learn how to get the kernel to recognize the display, and setup the framebuffer, so as to display beautiful images like this one:

![Desktop View](fb-test.png){: width="300" .shadow } 
_That's more colorful (running fb-test)_

## Setting up the no-screen VM
First, let's create a simple system based on the template Buildroot provides for [Qemu's `versatilepb`](https://www.qemu.org/docs/master/system/arm/versatile.html).

Get and extract Buildroot (2024.02.10 at the time of writing):
```bash
$ wget http://buildroot.org/downloads/buildroot-2024.02.10.tar.xz
$ xz -d buildroot-2024.02.10.tar.xz
$ tar -xvf buildroot-2024.02.10.tar
$ rm buildroot-2024.02.10.tar.xz
$ cd buildroot-2024.02.10
```

Now, let's use the defconfig for our target:

```bash
$ make qemu_arm_versatile_defconfig 
$ make
```

Once the build is done, we can boot it:
```bash
$ output/images/start-qemu.sh

...
Welcome to Buildroot
buildroot login: 
```

You can login with username `root`, and notice how no framebuffer device is available:
```bash
# ls /dev | grep fb0
#
```

## Enabling the display

### Manually
We need to enable **kernel** config options to have the kernel recognize the embedded PL110 LCD controller.\
Run:
```bash
$ make linux-menuconfig
```
Once in the menu, enable the following options (that Buildroot's default `qemu_arm_versatile_defconfig` does not enable):

```
Device Drivers
    -> Graphics support
        -> Direct Rendering Manager (XFree86 4.1.0 and higher DRI support) [y]
        -> Display Interface Bridges
            -> Display connector support [y]
        -> Display Panels
            -> ARM Versatile panel driver [y]
            -> support for simple Embedded DisplayPort panels [y]
            -> support for simple panels (other than eDP ones)
        -> DRM Support for PL111 CLCD Controller
        -> Console display driver support
            -> Framebuffer Console support [y]
            -> Map the console to the primary display device
```

Alternatively, you can search for these options and enable them:
```
DRM [y]
DRM_DISPLAY_CONNECTOR [y]
DRM_PANEL_ARM_VERSATILE [y]
DRM_PANEL_EDP [y]
DRM_PANEL_SIMPLE [y]
DRM_PL111 [y]
CONFIG_FRAMEBUFFER_CONSOLE [y]
CONFIG_FRAMEBUFFER_CONSOLE_DETECT_PRIMARY [y]
```

Save, exit and build.

```bash
$ make linux # if you only want to build the kernel
$ make       # to build everything
```

**Warning!** Your modifications will be lost when the build folder is cleaned!\
That's because Buildroot uses `output/build/linux-x.x.x/.config` to configure linux, and this folder is deleted by `make clean`.

### Automatically

You can use [this config patch](https://github.com/MachXNU/Engine-OS-on-Qemu/blob/main/enable-framebuffer.patch) that automatically enables these options.\
We can tell Buildroot to apply these patches on top of its configuration before building the kernel.

You can enable it in Buildroot by adding its path to the `BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES` variable in Buildroot.

## Conclusion

The system now boots with a working display !

![Desktop View](tux.png){: width="300" .shadow } 
_Tux!_

## Useful links
- [https://lukaszgemborowski.github.io/articles/minimalistic-linux-system-on-qemu-arm.html](https://lukaszgemborowski.github.io/articles/minimalistic-linux-system-on-qemu-arm.html)
- [https://www.qemu.org/docs/master/system/arm/versatile.html](https://www.qemu.org/docs/master/system/arm/versatile.html)
- [https://github.com/buildroot/buildroot/blob/master/board/qemu/arm-versatile/linux.fragment](https://github.com/buildroot/buildroot/blob/master/board/qemu/arm-versatile/linux.fragment)