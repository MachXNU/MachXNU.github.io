---
title: Verbose restore a 64 bits iPhone with checkm8
date: 2022-05-07 22:00:00 +0200
categories:  #[ TOP_CATEGORIE, SUB_CATEGORIE]
tags: []     # TAG names should always be lowercase
img_path : /2022-05-07-VERBOSE-RESTORE
---

## Introduction
In this post, you'll learn how to perform a verbose restore on a 64 bits iPhone using checkm8.
> While this should work on 32 bits devices too, I didn't have time to try it out yet. I will update these instructions or make a new blogpost once I have done it on such a device.
{: .prompt-info }

### Why do you want to do this ?
1. Maybe because you are curious and wonder what's happening behind the scenes. For sure, `idevicerestore` provides a interesting log for you to read, but you could actually get much more than this simple log.
2. When trying to dualboot an iPhone, I have trouble with the `apfs_invert()` method, and I thought comparing with a normal restore would help me diagnosing and hopefully solving my issues.

### Where to start ?
[@nyan_satan](https://twitter.com/nyan_satan) wrote a simple tutorial which you can find [here](https://nyansatan.github.io/verbose-restore/) decribing how to patch graphical routines. While this is definitely a great starting point, we also have to load our patch files, and this is what this tutorial is all about.

## The iOS restore process
> I am not an iOS expert, and what I'm saying might be wrong. Read it at your own risk. The following explanations are based on what I know and have understood from this project. I can't guarantee it's 100% (or even 1%) true. You'll have been warned.
{: .prompt-danger }

#### Update or erase ?
For the sake of this demo, we'll proceed to a so-called "update" restore. \
With iOS devices, there are 2 restore fashions : "update" and "erase".\
"Erase" restores perform some more operations on the disk (they wipe user data), and unfortunately I've not been able to perform such restores with UI routines disabled. On the other hand, "update" restore seems more cooperative, so that's the restore style we'll use. 

#### What happens during a restore ?
We'll start with a device placed in recovery mode.\
At this stage, `iBSS` and `iBEC` are already loaded and the device is waiting for the next boot images, including (but not limited to) `kernelcache`, a  `ramdisk` and its associated `trustcache`, `devicetree`, etc.\
But here, we don't have to care that much about what has to be sent, because `idevicerestore` already sends all the necessary files for us in the correct order.\
Our role "only" boils down to giving `idevicerestore` a valid IPSW file to work with.\
This sounds simple, but since we are patching things out, we're breaking the chain of trust at some point, and restore will thus fail if we don't do anything.

#### What do we have to patch ?
During a restore, the restore software (either iTunes, `idevicerestore`, `futurerestore` or whatever) sends a ramdisk, among other files. \
An IPSW contains 2 ramdisks, one for each restore type (their exact name can be found on [The iPhone WIki](https://www.theiphonewiki.com), in the Firmware Keys section).\
Inside the ramdisks, at path `/usr/local/bin` is a Mach-O executable which effectively performs the restore operations on the device side (the PC also plays its role, but the iPhone's restore code is located in this executable). \
For "erase" variants, this executable is called `restored_external` while in "update" variants, it's called `restored_update`.\
This is the executable we need to patch : it's responsible for eprforming the restore but also handles the UI part (displaying the Apple logo and the progress bar)

#### Some more complications
Of course, it would be too easy to just patch this executable, repack the IPSW and restore it, right ?\
That's because, when patching `restored_update`, we altered its signature. Of course, we will fakesign it with `ldid2` but the kernel, if unpatched, will reject its signature.\
So we need to patch the kernel (we'll see how to do this later on). But patching the kernel alters its signature, so `iBEC` won't load it. We thus need to patch both `iBSS` and `iBEC` and this brings us back to **checkm8** !\
Patching signature validations if what will make everything possible.\

> Since we are flashing patched `iBSS` and `iBEC` to our device, these images won't be validated if SecureROM is not patched. In other words, after this restore, you'll have to tether boot your device.\
However, this can be easily reverted by restoring a normal IPSW : this will falsh valid boot images, which will be booted untethered.
{: .prompt-danger }

#### Credits
As I mentionned earlier, I got the idea to do this project thanks to [@nyan_satan](https://twitter.com/nyan_satan)'s blog post.\
I also followed [@mcg29_](https://twitter.com/mcg29_)'s [dualboot guide](https://dualbootfun.github.io/dualboot/) for learning how to load a custom bootchain.

## Requirements
Here are the tools you need for this project (I recommend you copy the compiled executables to `/usr/local/bin` on your Mac to always have them in your path) :
* [img4lib](https://github.com/xerub/img4lib) aka `img4` by @xerub
* [kairos](https://github.com/dayt0n/kairos) by @dayt0n
* [Kernel64Patcher](https://github.com/Ralph0045/Kernel64Patcher) by @Ralph0045
* ldid2 (install it with `brew install ldid2`)
* [asr64_patcher](https://github.com/exploit3dguy/asr64_patcher) by @md
* idevicerestore (compile everything with [this](https://gist.github.com/matteyeux/d7d8041a41ee8d664aaf5c3b99556ada) by @matteyeux)

## Before you begin
If you're not really familiar with iOS boot/restore processes, I recommend you first do the following :
1. Restore a normal IPSW with `idevicerestore` and read the log
2. Read the [dualboot guide](https://dualbootfun.github.io/dualboot/) by [@Ralph0045](https://twitter.com/Ralph0045) and [@Ralph0045](https://twitter.com/mcg29_) and try it yourself 
3. Verbose boot your iPhone. It's an absolutely basic thing you MUST master well. It's trivial, but do it by hand, not with checkra1n :)

## Let's get started
### The general idea of the operation
1. Extract a release IPSW (the latest one, which can be restored without blobs). While it is theoretically possible to restore with `futurerestore` to an unsigned version, I didn't try it yet and can't confirm it works out of the box.
2. Patch `iBSS` and `iBEC` to load custom images ; and replace them in the original IPSW.
3. Patch the `kernelcache` to disable AMFI and replace it in the original IPSW.
4. Extract the "update" ramdisk and locate `restored_update`.
5. Patch UI routines in `restored_update`, restore its entitlements and fakesign it.
6. Patch `asr` to force image validation.
7. Pack back the ramdisk.
8. Pack back the IPSW and restore it.

> You are free to organise the project's folders as you wish, so be careful since your commands may vary from mine.\
Copy-pasting is never a good idea, and you won't learn anything.\
As for me, I've created a `restore` folder for all the stuffs with inside, the stock IPSW extracted in a folder called `stock`, and the modded IPSW in a folder called `modded`.
{: .prompt-info }

### Patching `iBSS` and `iBEC`
If you have already verbose booted your iPhone, this part should look familiar to you.\

> I'm using the kbag method to decrypt `iBSS` and `iBEC`. Thanks to this, I don't rely on keys from The iPhone Wiki.
{: .prompt-info }

For `iBSS` (for example), first extract its kbag
```bash
img4 -i iBSS.*.RELEASE.im4p -b
```
The kbag is the first line in the output.\
Then, place your iPhone in DFU mode, exploit it with `./ipwndfu -p` but don't remove sigchecks yet ! Instead, run :
```bash
./ipwndfu --decrypt-gid=<kbag>
```
This is the decrypted kbag (aka `dkbag`). We can now use it to decrypt `iBSS` :
```bash
img4 -i iBSS.*.RELEASE.im4p -o ibss.raw -k <dkbag> 
```

Then patch it with `kairos` :
```bash
kairos ibss.raw ibss.pwn
```
And pack it back to `im4p` :
```bash
img4 -i ibss.pwn -o ibss.im4p.pwn -A -T ibss
```

> Note that unlinke
{: .prompt-warning }
