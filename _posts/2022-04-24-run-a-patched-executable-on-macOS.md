---
title: Run a patched executable on macOS
date: 2022-04-24 12:00:00 +0200
categories:  #[ TOP_CATEGORIE, SUB_CATEGORIE]
tags: []     # TAG names should always be lowercase
img_path : /2022-04-24-MAC-CODESIGN
---

## Introduction
In the [previous article](https://machxnu.github.io/posts/13-00-PATCH/), we discussed how to patch an executable to redirect execution flow.\
The problem is that if you try to run this on your Mac right after patching, the system will refuse to execute it : `zsh killed`\
Don't forget to add executable permissions first with `chmod +x ./age` though.\
Afterwards, you get :
```bash
$ ./patcher.py ./age
[i] Replacing instruction at offset 0x3f04
Patched file created
$ chmod +x ./age
$ ./age_patched
zsh: killed     ./age_patched
```

## Identifying the problem
It is pretty straightforward, and after having a look at the Console app, we can see that the problem comes from the executable's **signature**.\
Just to be sure, we can run :
```bash
$ codesign -v ./age_patched 
./age_patched: invalid signature (code or signature have been modified)
In architecture: arm64
```

## Creating a code signing certificate
We have to re-sign our executable, but first we must create a code signing certificate for that.\
This procedure has been described countless times online to sign `gdb` on macOS, so if you have trouble with this step, there are plenty of alternative tutorials available.\
1. Open `Keychain.app`, go to `Certificate Assistant > Create a certificate...`
![Desktop View](1.png){: .shadow }
2. Select `Self-Signed Root`, `Code Signing`
![Desktop View](2.png){: .shadow } 
3. Unlock `System` keychain
![Desktop View](3.png){: .shadow } 
4. Move the certificate from `Login` to `System`\
If you have trouble doing this, try a copy/paste + delete original
![Desktop View](4.png){: .shadow } 
5. Right click on your certificate, `Get info` and expand the `Trust` triangle. In the drop-down menu, choose `Always trust`
![Desktop View](5.png){: .shadow } 
6. Close the window and authenticate.\
If you've done everything correctly, there is a `+` sign next to your certificate.
![Desktop View](6.png){: .shadow } \
Our certificate is ready to use !

## Signing the patched executable
Now you can sign your patched executable :
```bash
$ codesign -fs MachXNU ./age_patched
./age_patched: replacing existing signature
```
You can now execute the (patched and) signed binary :
```bash
$ ./age_patched 
How old are you ? 10
You are an adult
```

## Conclusion
Creating a code signing certificate and signing the patched binary is enough to make macOS execute it !
