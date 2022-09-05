---
title: Fulling theming macOS Big Sur
date: 2022-09-05 14:00:00 +0200
categories:  #[ TOP_CATEGORIE, SUB_CATEGORIE]
tags: []     # TAG names should always be lowercase
img_path : /2022-09-05-MACOS-THEMING
---

## Introduction
Everyone with a jailbroken has probably already themed their iPhone, right ?\
Well, actually, you don't even need to be jailbroken to enjoy alternative icons on iOS, but theming has been a long tradition in the jailbreak community.

![Desktop View](iOS.png){: width="300" .shadow } 
_Felicity Pro and Dotto+ on iPhone 11_

So, why not doing the same on macOS ? \
Well, mostly because, just like iOS, it's not possible on stock. While you can indeed change thrid-party apps icons easily, more advanced theming (like system apps icons or notification badges) cannot be customized on stock.

In this article, I'm going to describe the different techniques one can use to achieve full theming on macOS Big Sur. This may work as well on Monterey (and Ventura ?) but I haven't tested it yet.\
Here is a little preview of what we will be achieving :
![Desktop View](1.png){: .shadow } 

## Where to get icons for macOS ?
If you don't want to make custom icons yourself, you can visit [macosicons.com](https://macosicons.com). They have thousands of incredible icons and you'll most probably find one you like.\
If you can't find what you're looking for, you'll have to make icons yourself. To edit a picture online, you can use [lunapic.com](https://www7.lunapic.com/editor/) to edit images, then turn them into .icns files using [Image2icon](https://apps.apple.com/fr/app/image2icon-make-your-icons/id992115977?l=en&mt=12) available for free on the Mac AppStore.

## Disabling SIP and SSV
To achieve our goals, we have to disable SIP (System Integrity Protection) and SSV (Signed System Volume). These are system protections built in macOS and disabling them *may* alter your system security and leave your Mac more vulnerable to malware.

> Please do NOT do this is you don't know what you're doing !\
I'm not responsible for any problem that may result from doing this.
{: .prompt-danger }

Boot into Recovery Mode, open the terminal and run :\
```bash 
csrutil disable
csrutil authenticated-root disable
```
Then reboot to macOS.\
If your Mac fails to boot (there is no real reason this can happen if you didn't mess anything, but that happened to me once), just reinstall macOS. You won't lose your data if you don't erase your main disk. As usual, having a backup is the safest way to prevent data loss (which can always happen...).\
Also note that enabling **verbose boot** can help diagnosing issues during boot if you encounter any.

Once in macOS, check if everything is disabled correctly :\
run ```csrutil status```, you should see it's disabled.\
Same, run ```csrutil authenticated-root status```, it should be disabled too.

## Changing third-party apps icons
> Editing third-party apps icons does **NOT** require SIP and/or SSV to be disabled, so you can keep them on if you're just planning to theme third party apps.
{: .prompt-tip }

Go to Finder -> Applications and select the app you want to theme.\
Then, you can either press `Command + I` or go right-click -> `Get Info`.\
Finally, drag and drop your new icon into the icon slot :
![Desktop View](3.png){: width="300" .shadow }
_Drop here_  
This is trivial and has already been discussed countless times online, so I won't go into any further details here.

## Changing System apps icons
This is normally not possible because the System volume is mounted as read-only. Unfortunately, it's not possible to remount it directly as read-write (it used to be possible on Catalina and earlier) and to do it, it's necessary to disable SIP and SSV, as we did earlier.

### Which volume should we mount ?
Open `Disk Utility` and look for `Macintosh HD`. Note down the device identifier (here, it is `/dev/disk3s1`).
![Desktop View](4.png){: .shadow }
> It's normal if you don't have the "System snapshot mounted" message. It appears for me because I've already done all the steps described further down the article.
{: .prompt-info }

Then, create a mountpoint :
```bash
mkdir /tmp/mount
```

and finally, mount your disk here
```bash
sudo mount -w -t apfs /your/identifier /tmp/mount
```
![Desktop View](5.png){: .shadow }

### Editing the icons
Then we can navigate to `/tmp/mount/System/Applications/` where System apps are stored. Now, this location is readable and writable.\
You may now change any icon you want using the method decsribed earlier for third party apps (using drag and drop).
![Desktop View](6.png){: .shadow }

Then, I strongly recommend you test your edits by performing the steps described below in the section "Saving changes".

## Modifying notification badges
Do you seriously find these red notification badges pretty ?
![Desktop View](7.png){: .shadow }
_Sorry, but that's ugly_

I'm ususally a huge fan of "stock" visuals, but these huge red dots look really ugly, especially if you've opted, like me, for a certain color scheme. These badges can look horrible.\
So, let's change them ! More precisely, we're gonna change the badges *color*.

With your System volume mounted as read-write (see the "Changing System apps icons" section above on how to achieve this), navigate to `/tmp/mount/System/Library/CoreServices/Dock.app/Contents/Resources`.\
Once you're there, find 2 files called `statuslabel.png` and `statuslabel@2.png`.

Just backup them and replace them with custom icons (make sure to keep the same resolution and the **exact same filename**). 
![Desktop View](8.png){: .shadow }

Save your changes (see section below), reboot and you'll see nice badges with the color that suits best to your theme :
![Desktop View](9.png){: .shadow }
_There is a badge on the Reminders app_

## Modifying the Finder icon
Same as before, mount your System volume as read-write and navigate to `/tmp/mount/System/Library/CoreServices/Dock.app/Contents/Resources`.\
Then, just replace `finder.png` and `finder@2.png` by yours.
![Desktop View](10.png){: .shadow }
Again, don't forget to backup the original files, save your changes and reboot.

## Modiying the calendar icon
Because yes, there is a big quirck with the Calendar app : its icon is generated at runtime, because it changes every day...\
Trying any of the above methods will not work (you can try yourself, but I already did it for you...).

You could force the system to use a static icon (editing the Calendar app's `Info.plist`) but the icon won't change over time, which is not satisfying.

So, we have to understand how the Calendar app changes its icon. Reversing the executable in Hopper was useless, same when reversing its Docktile plugin (`/System/Applications/Calendar.app/Contents/Resources/Calendar.docktileplugin`).

Searching [online](https://logos.fandom.com/wiki/Calendar_(macOS)) suggests the existence of an `App-empty.icns` file that we could replace to achieve what we want.\
However, while this has already been discussed [here](https://forums.macrumors.com/threads/change-calendar-icon.1900924/), this file can longer be found where we expect it to be.\
Fortunately, the terminal can help us, and running ```sudo find / -name "App-empty.icns" ``` shows interesting results :
![Desktop View](11.png){: .shadow }

Let's navigate to this path, but using our read-write permissions to be able to edit the file !\
So, we'll go `/tmp/mount/System/Library/PrivateFrameworks/CalendarUIKit.framework/Versions/A/Resources/`

Once there, you can backup and replace the `App-empty.icns` file !
![Desktop View](12.png){: .shadow }
Once again, save your changes and reboot. The calendar icon will have changed.
![Desktop View](13.png){: width="100" .shadow }
_Cool, isn't it ?_

## Saving changes
We now have to make a new snapshot with your last changes and tell the system to use this snapshot to boot.\
Skipping this step will result in all your changes being lost.

```bash
sudo bless --folder /tmp/mount/System/Library/CoreServices --bootefi --create-snapshot
```

Then, we have to flush the icon cache, and force the system to regenerate a new one.
```bash
sudo rm -rfv /Library/Caches/com.apple.iconservices.store > /dev/null 2>&1
sudo find /private/var/folders/ -name com.apple.dock.iconcache -exec rm -rf {} \; > /dev/null 2>&1
sudo find /private/var/folders/ -name com.apple.iconservices -exec rm -rf {} \; > /dev/null 2>&1
```
Finally, unmount the disk
```bash
sudo umount -f /tmp/mount
```
and **reboot** !

You should see your changes saved and in action !\
Congratulations ! 🎉
