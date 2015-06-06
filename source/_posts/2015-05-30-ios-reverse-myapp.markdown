---
layout: post
title: "ios reverse myapp"
date: 2015-05-30 22:26:50 +0800
comments: true
categories: [IOS, reverse]
---
## reverseMyApp
### download tools and install
[class-dump-z](https://code.google.com/p/networkpx/wiki/class_dump_z)
[Charles](http://www.charlesproxy.com/download/)

### Edit Scheme
Select `Run` panel, then in `info` tab, select `Release` under `Build Configuration`;

### enter Simulator App directory
you can use `SimPholders2`

<!--more-->

### dump classes
1. check out what Frameworks the bundle links to
    `otool -L app`
2. class dump
    `class-dump-z app > app.txt`
3. find "manager, "shared" or "store"

### print plist
    plutil -p Info.plist

### NSUserDefaults directory
    {App Directory}/Library/Preferences/{Bundle Identifier}.plist
    open Library/Preferences/{Bundle Identifier}.plist

### open finder in current directory
    open .

### Keychain Access
    Keychain  is encrypted with the help of the user’s password, which generally is the simple 4-digit numeric passcode.

    1. Encrypt the data
        encrypting the data using Apple’s Common Crypto APIs found in the Security Framework
    2. Do NOT hardcode your encryption key to the app
        You need to make a unique encryption key for the device
    3. Be aware of your methods and how an attacker can use them
    4. Question yourself: Do you need to store it?

## Network Penetration Testing and Mapping

### strings
    strings app | grep http
http://version1.api.memegenerator.net/Generator_Select_ByUrlNameOrGeneratorID


## LLDB
    gdb -q  
    (lldb) process attach --name "Meme Collector" --waitfor

    //make breakpoints
    (lldb)b viewDidLoad

    //show breakpoints address
    (lldb)br l

    //continue
    (lldb)c

![](http://cdn5.raywenderlich.com/wp-content/uploads/2013/07/Terminal-GDB.png)

    call [[MoneyManager sharedManager] purchaseCurrency]
   












