---
title: Run Android apps in a debugger without root
date: 2025-04-11 13:00:00 +0100
categories:  #[ TOP_CATEGORIE, SUB_CATEGORIE]
tags: []     # TAG names should always be lowercase
---

## Introduction

In this article, I will explain how to run a compiled third-party Android app in a debugger like `lldb` on a non-rooted device.

## Prerequisites

Install [Android Studio](https://developer.android.com/studio) and download the SDK and NDK.\
I won't go into much details about how to do this.

You can check that the installation was successfull if :
- you can run `adb`
- you have the NDK installed (on macOS with default settings, it will be in `/Users/$(whoami)/Library/Android/sdk/ndk`)
- you can find `lldb-server` in `$NDK-PATH/*/toolchains/llvm/prebuilt/*/lib/clang/*/lib/linux/aarch64/lldb-server`

## Make sure your .apk is debuggable

You can follow [this tutorial](https://gist.github.com/amahdy/87041554f62e384bee5766a958fd4f9a) to make a release `.apk` debuggable

## A few words before continuing

Before going any further, I want to explain the challenges that we are facing when trying to run our debugger on a non-rooted device.\
**NB** : I am running this experiment on a Meta Quest, and these devices have special (annoying) read-write issues that may not happen with other Android devices.

On Meta Quest, we can sideload our `lldb-server` to a writable place, like `/sdcard/Documents`. Let's try this :

```
$ adb push lldb-server /sdcard/Documents/lldb-server
$ adb shell /sdcard/Documents/lldb-server
/system/bin/sh: /sdcard/Documents/lldb-server: can't execute: Permission denied
$ adb shell ls -l /sdcard/Documents/lldb-server
-rw-rw---- 1 root everybody 49731928 2025-04-11 20:16 /sdcard/Documents/lldb-server
$ adb shell chmod 777 /sdcard/Documents/lldb-server
$ adb shell ls -l /sdcard/Documents/lldb-server
-rw-rw---- 1 root everybody 49731928 2025-04-11 20:16 /sdcard/Documents/lldb-server
```
Indeed, we do not have execution rights. And we can't add them neither with `chmod`.

To be able to debug our app, the debugger needs to run with the same level of priviledge than the app. So let's try :
```
$ adb shell run-as com.bundle.identifier /sdcard/Documents/lldb-server
run-as: exec failed for /sdcard/Documents/lldb-server: Permission denied
$ adb shell run-as com.bundle.identifier ls /sdcard/Documents
ls: /sdcard/Documents: Permission denied
```

See, the `com.bundle.identifier` user does not even have right to read `/sdcard/Documents`.\
But then, where can our app read/write/execute ?

```
$ adb shell run-as com.bundle.identifier pwd
/data/user/0/com.bundle.identifier
$ adb shell run-as com.bundle.identifier touch test
$ adb shell run-as com.bundle.identifier chmod 777 test
$ adb shell run-as com.bundle.identifier ls -l test
-rwxrwxrwx 1 u0_a169 u0_a169 0 2025-04-11 20:18 test
$ adb shell run-as com.bundle.identifier rm test
```

So, sideloading our `lldb-server` to this location should work, right ? Well, yes, except we can't :
```
$ adb push lldb-server /data/user/0/com.bundle.identifier/lldb-server
adb: error: stat failed when trying to push to /data/user/0/com.bundle.identifier: Permission denied
```

We can solve this issue, using piping :
```
$ tar cf - lldb-server | adb shell 'run-as com.bundle.identifier tar xf - '
```

Now, we can _finally_ execute `lldb-server` :
```
$ adb shell run-as com.bundle.identifier ./lldb-server
Usage:
  ./lldb-server v[ersion]
  ./lldb-server g[dbserver] [options]
  ./lldb-server p[latform] [options]
Invoke subcommand for additional help
```

## Debugging the app
Now, it's time to debug our app.\
Let's launch `lldb-server` :
```
$ adb shell run-as com.bundle.identifier ./lldb-server platform --listen "*:10086" --server
```

Forward the port with `adb` :
```
$ adb forward tcp:10086 tcp:10086
```

Start the app (you can also launch it directly via your graphical interface on the device) :
```
$ adb shell am start -n com.bundle.identifier/com.your.activity
```

Get the app's PID :
```
$ adb shell ps -A | grep com.bundle.identifier
```

Get your device ID :
```
$ adb devices
```

Launch `lldb` on your host:
```
$ lldb
(lldb) platform select remote-android
(lldb) platform connect connect://YOUR-DEVICE-ID:10086
(lldb) process attach --pid XXXX
```

(Optional) Ignore signals SIGXCPU and SIGPWR :
```
(lldb) process handle SIGXCPU -n false -p true -s false
(lldb) process handle SIGPWR -n false -p true -s false
```
Source : [StackOverflow](https://stackoverflow.com/questions/70018886/how-to-stop-signal-sigpwr-power-fail-restart-while-debugging-xamarin-android-c)