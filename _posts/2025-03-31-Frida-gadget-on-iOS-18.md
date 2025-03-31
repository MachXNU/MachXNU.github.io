---
title: Frida Gadget/Objection on iOS 18
date: 2025-03-30 17:00:00 +0100
categories:  #[ TOP_CATEGORIE, SUB_CATEGORIE]
tags: []     # TAG names should always be lowercase
img_path : /2025-03-31-FRIDA-iOS
---

## Introduction
Running Frida gadget on (jailed) iOS 18 can be suprisingly more difficult that expected.

In this article, I am providing a full method to get Frida Gadget running on such a device.

## Prerequisites
- a device running iOS 18
- a Mac
- Xcode (can be Xcode 15+)
- Sideloadly
- [Objection](https://github.com/sensepost/objection)

## Steps
### 1) Get a decrypted IPA
You can get a decrypted IPA for your app at [armconverter.com/decryptedappstore](https://armconverter.com/decryptedappstore/us).

![](decrypted-app-store.png)

### 2) Get a signing identity and an `embedded.mobileprovision`

In Xcode, create a sample app for iOS and run it on your device.

- You can view your signing identity with: 
```
% security find-identity -p codesigning -v                          
  1) 85719B2FB2XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX "Apple Development: xxxxxxx@xyz.com (YYYXXXXXXX)"
     1 valid identities found
```

- You can extract the `embedded.mobileprovision` from the build result from Xcode _(for example `/Users/xxx/Library/Developer/Xcode/DerivedData/SampleApp-xxxxxxxxxxxxxxxx/Build/Products/Debug-iphoneos/SampleApp.app/embedded.mobileprovision`)_

You can get more details on the [Objection GitHub page](https://github.com/sensepost/objection/wiki/Patching-iOS-Applications).

### 3) Patch the app

Follow the instructions on [Objection wiki for iOS](https://github.com/sensepost/objection/wiki/Patching-iOS-Applications) to prepare a patched `.ipa`.

This should give you a fresh `.ipa` like this:

![](objection-patched.png)

### 4) Install the app

Unfortunately, as of iOS 17+, [we can no longer use ios-deploy to launch apps](https://github.com/ios-control/ios-deploy/issues/588). So we'll not use this tool at all.

Instead, we can just use Sideloadly !\
You can take note of the bundle identifier here, and change it if needed.

![](sideloadly.png)

### 5) Launch the app
If you just try to launch the app by tapping its icon on the home screen, it will just crash.

Run this command to start the app paused:
```
xcrun devicectl device process launch --device 00008120-<identifier> --start-stopped com.bundle.identifier
```

You can get your device identifier from `Xcode > Window > Devices and Simulators`.

The app should launch and freeze.

Next go to `Xcode > Debug > Attach to process` and find your app.

![](attach.png)

You should see `Frida: Listening on 127.0.0.1 TCP port 27042` on the console.

![](console.png)

### 6) Connect to Frida

We need to use the networked mode.\
On a terminal window, run:
```
pymobiledevice3 usbmux forward 27042 27042
```

and in another:

```
objection -N -h 127.0.0.1 -p 27042 explore
```

![](connected.png)

Congrats ! 🥳 You are connected !