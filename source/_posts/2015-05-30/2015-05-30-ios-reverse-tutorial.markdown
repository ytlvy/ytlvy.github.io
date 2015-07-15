---
layout: post
title: "ios reverse tutorial"
date: 2015-05-30 22:26:41 +0800
comments: true
categories: [IOS, reverse]
---
## setting platform
### jailbreak
### add sources
    1. apt.178.com
    2. apt.weiphone.com

### install tools
    openssh
    BigBoss Recommended tools
    MobileTerminal

### ssh login
    user: root pwd:alpine
    change password GJJ123456

### update
    apt-get update
    apt-get upgrade

<!--more-->

## get class information
### Dumping class information
>user app diretory
>/private/var/mobile/Containers/Bundle/Application/

    scp class-dump-z root@192.168.1.101:/var/root
    cp class-dump-z /usr/bin
    chmod 755 /usr/bin/class-dump-z

    Clutch -i
    Clutch -d com.tencent.mipadqq
    cd /private/var/mobile/Documents/Dumped

### Keychain-Dumper
    //赋予Keychain数据库可读权限
    cd /private/var/Keychains/ 
    chmod +r keychain-2.db
    keychain_dumper > keychain-export.txt 

## Cycript




### 
1. The Text Section: This section is largely for read-only data,For example, 
    it includes the source code section, the C-type strings section, the constants, etc
2. The Data Section: This section is largely for writable data from the program. 
    This includes the bss section for static variables, the common section for global variables, etc
```bash
//print out Meme Collector’s Mach-O header
otool -h Meme\ Collector

otool -l Meme\ Collector
//find sectname __classname section |  __objc_classname

//displays the global offset of where the strings are in the binary
strings -o Meme\ Collector

//The nm command displays information regarding the symbol table
nm Meme\ Collector | grep aSecretFunction

```
![](http://cdn5.raywenderlich.com/wp-content/uploads/2013/07/Terminal-Binary-Sections-700x437.png)

### App Extension 脱壳
> 1. 从app store下载的app和app extension是加过密的
> 2. iPhone applications的解密办法: dumpdecrypted, app脱壳开源工具.原理是：将应用程序运行起来（iOS系统会先解密程序再启动）
> ，然后将内存中的解密结果dump写入文件中，得到一个新的可执行程序
> 3. iPhone app extensions的特别之处, dumpdecrypted 无法实现对iPhone app extensions的脱壳
> > app extension虽是独立进程，但不可独立运行
> > app extension的进程中，写权限被严格控制
> 4. 修改版dumpdecrypted：
> > 1. 本地编译好 dumpdecrypted.dylib
> > 2. 指定作用的Extension Bundle
> > 3. 将 dumpdecrypted.plist 和 dumpdecrypted.dylib 拷贝至越狱机的 
> > /Library/MobileSubstrate/DynamicLibraries/ 下
> > 4. 利用系统相册启动微信的Share Extension

```bash
//查看加密
otool -l binary_name | grep crypt

// 修改版dumpdecrypted
vim dumpdecrypted.plist
{
    Filter = {
        Bundles = ("com.tencent.xin.sharetimeline");
    };
}

//当微信的Share Extension被启动时，解密插件自动工作。值得注意的是，
//如果你的越狱机是armv7架构，那么也就只dump armv7那部分；如果越狱机是arm64架构，
//那么也就只dump arm64那部分。得到干净的dump结果执行下面
$ lipo -thin armv7 xxx.decrypted -output xxx_armv7.decrypted //或者
$ lipo -thin armv64 xxx.decrypted -output xxx_arm64.decrypted
```




### iOS应用逆向常用工具
    - Reveal
    - Cycript
    - Class-dump
    - Keychain-Dumper
    - gdb
    - iNalyzer
    - introspy
    - Fishhook
    - removePIE
    - IDA pro or Hopper
    - snoop-it
    - iDB
    - Charles
    - SSL Kill Switch

#### 常用的命令和工具
    - ps           ——显示进程状态，CPU使用率，内存使用情况等
    - sysctl       ——检查设定Kernel配置
    - netstat     ——显示网络连接，路由表，接口状态等
    - route        ——路由修改
    - renice       ——调整程序运行的优先级
    - ifconfig    ——查看网络配置
    - tcpdump   ——截获分析网络数据包
    - lsof           ——列出当前系统打开的文件列表，别忘记一切皆文件，包括网络连接、硬件等
    - otool ①     ——查看程序依赖哪些动态库信息，反编代码段……等等等等
    - nm ②        ——显示符号表
    - ldid ③      ——签名工具
    - gdb          ——调试工具
    - patch       ——补丁工具
    - SSH         ——远程控制

```bash
    //可查看可执行程序都链接了那些库
    otool  -L WQAlbum 
    //可以反编译WQAlbum的__TEXT__段内容, 截前10行：
    otool -tV WQAlbum |head -n 10

    //nm，显示程序符号表，用我自己的应用程序私人相册现身说法一下
    nm -g WQAlbum  （ -g 代表 global） 

    //ldid，是iPhoneOS.platform提供的签名工具，我们自己编译的程序需要签上名才能跑在iPhone/iPad上，使用方法
    export CODESIGN_ALLOCATE=/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/usr/bin/codesign_allocate
    ldid -S helloworld
```


### 裸奔app的安全隐患
    - 任意读写文件系统数据
    - HTTP(S)实时被监测
    - 重新打包ipa
    - 暴露的函数符号
    - 未加密的静态字符
    - 篡改程序逻辑控制流
    - 拦截系统框架API
    - 逆向加密逻辑
    - 跟踪函数调用过程（objc_msgSend）
    - 可见视图的具体实现
    - 伪造设备标识
    - 可用的URL schemes
    - runtime任意方法调用
    - ……



#### cycript
##### Objective-C objects from addresses
```
    var p = new Instance(0x8614390)
```

##### Getting ivars
```
    //*varName
    cy# *controller

    function tryPrintIvars(a){ var x={}; for(i in *a){ try{ x[i] = (*a)[i]; } catch(e){} } return x; }
    cy# tryPrintIvars(a)
```


##### Getting methods
```
function printMethods(className) {
  var count = new new Type("I");
  var methods = class_copyMethodList(objc_getClass(className), count);
  var methodsArray = [];
  for(var i = 0; i < *count; i++) {
    var method = methods[i];
    methodsArray.push({selector:method_getName(method), implementation:method_getImplementation(method)});
  }
  free(methods);
  free(count);
  return methodsArray;
}

cy# printMethods("MailboxPrefsTableCell")
```


##### Get methods matching particular RegExp
```
function methodsMatching(cls, regexp) { return [[new Selector(m).type(cls), m] for (m in cls.messages) if (!regexp || regexp.test(m))]; }
cy# methodsMatching(NSRunLoop, /forKey:$/)
```

##### Getting class methods
```
cy# NSRunLoop->isa.messages['currentRunLoop'] = ...
```


##### Replacing existing Objective-C methods func.call(self, arg1, arg2, arg3, ...)
```
cy# original_NSRunLoop_description = NSRunLoop.messages['description'];
cy# NSRunLoop.messages['description'] = function() { return original_NSRunLoop_description.call(this).toString().substr(0, 80)+", etc."; }
cy# [NSRunLoop currentRunLoop]
"<CFRunLoop 0x205ee0 [0x381dbff4]>{locked = false, wait port = 0x1303, stopped = , etc."

cy# original_SpringBoard_menuButtonDown = SpringBoard.messages['menuButtonDown:']
cy# SpringBoard.messages['menuButtonDown:'] = function(arg1) {original_SpringBoard_menuButtonDown.call(this, arg1);}
```

##### Load frameworks
```
function loadFramework(fw) {
  var h="/System/Library/",t="Frameworks/"+fw+".framework";
  [[NSBundle bundleWithPath:h+t]||[NSBundle bundleWithPath:h+"Private"+t] load];
}
```

##### Include other Cycript files
```
localhost:~ mobile$ cycript -p SpringBoard main.cy
localhost:~ mobile$ cycript -p SpringBoard

// include other .cy files
function include(fn) {
  var t = [new NSTask init]; [t setLaunchPath:@"/usr/bin/cycript"]; [t setArguments:["-c", fn]];
  var p = [NSPipe pipe]; [t setStandardOutput:p]; [t launch]; [t waitUntilExit]; 
  var s = [new NSString initWithData:[[p fileHandleForReading] readDataToEndOfFile] encoding:4];
  return this.eval(s.toString());
}
```

##### Using NSLog
```
cy# NSLog_ = dlsym(RTLD_DEFAULT, "NSLog")
0x31451329
cy# NSLog = function() { var types = 'v', args = [], count = arguments.length; for (var i = 0; i != count; ++i) { types += '@'; args.push(arguments[i]); } new Functor(NSLog_, types).apply(null, args); }
{}
cy# NSLog("w ivars: %@", tryPrintIvars(w))
//If you are attached to a process, the output is going to be in the syslog:


```

##### Using CGGeometry functions
```
function CGPointMake(x, y) { return {x:x, y:y}; }
function CGSizeMake(w, h) { return {width:w, height:h}; }
function CGRectMake(x, y, w, h) { return {origin:CGPointMake(x,y), size:CGSizeMake(w, h)}; }

```

##### Writing Cycript output to file
```
[[someObject someFunction] writeToFile:"/var/mobile/cycriptoutput.txt" atomically:NO encoding:4 error:NULL]
```
You can use this, for example, to get a dump of SpringBoard's view tree
```
iPhone:~$ cycript -p SpringBoard
cy# [[UIApp->_uiController.window recursiveDescription] writeToFile:"/var/mobile/viewdump.txt" atomically:NO encoding:4 error:NULL]
```


##### Printing view hierarchy
```
cy# ?expand
expand == true
cy# UIApp.keyWindow.recursiveDescription
```

##### Cycript scripts
Custom shell function that loads a cycript file:
```
cyc () { cycript -p $1 /var/root/common.cy > /dev/null; cycript -p $1; }
Usage: cyc ProcessName
```
Add this to /etc/profile.d/cycript.sh to make it available in all sessions.

##### Weak Classdump (Cycript based class-dump)
```
root# cycript -p Skype weak_classdump.cy; cycript -p Skype
 'Added weak_classdump to "Skype" (1685)'
 cy# UIApp
 "<HellcatApplication: 0x1734e0>"
 cy# weak_classdump(HellcatApplication);
 "Wrote file to /tmp/HellcatApplication.h"
 cy# UIApp.delegate
 "<SkypeAppDelegate: 0x194db0>"
 cy# weak_classdump(SkypeAppDelegate,"/someDirWithWriteAccess/");
 "Wrote file to /someDirWithWriteAccess/SkypeAppDelegate.h"
 root# cycript -p iapd weak_classdump.cy; cycript -p iapd
 'Added weak_classdump to "iapd" (1127)'
 cy# weak_classdump(IAPPortManager)
 "Wrote file to /tmp/IAPPortManager.h"
 root# cycript -p MobilePhone weak_classdump.cy; cycript -p MobilePhone
 'Added weak_classdump to "MobilePhone" (385)'
 #cy weak_classdump_bundle([NSBundle mainBundle],"/tmp/MobilePhone")
 "Dumping bundle... Check syslog. Will play lock sound when done."
```


### Yahoo Weather app

#### install
```bash
    dpkg -i cycript.deb
```

#### use

1. ps aux | grep "weather"
2. cycript -p 1710
3. var app = [UIApplication sharedApplication] && UIApp(default val)
4. var appdelegate = app.delegate
5. UIApp.windows
6. UIApp.keyWindow.rootViewController
7. printMethods(YWAppDelegate)  //printMethods see defintion above
8. [[[[[UIApplication sharedApplication] keyWindow] subviews] objectAtIndex:0] nextResponder].topViewController
9. printMethods(YWMainViewController)
10. [[[[[[UIApplication sharedApplication] keyWindow] subviews] objectAtIndex:0] nextResponder].topViewController userDidRequestUpdate]
11. tryPrintIvars([[[[[UIApplication sharedApplication] keyWindow] subviews] objectAtIndex:0] nextResponder].topViewController)

### Method Swizzling Using Cycript
1. cy# Appdelegate->isa.messages
2. Appdelegate.messages
3. ViewController.messages
4. ViewController.messages['validateLogin'] = function(){return true}//Swizzling
5. [UIApp.keyWindow.rootViewController.visibleViewController pushLoginPage]

### IOS fileSystem
1. user `mobile`, `ps aux`
2. cydia dufault user `root` password `alpine`
3. /Applications   ---------   system preinstalled or cydia installed
4. /private/var/mobile/Applications
5. /var/mobile/Library/AddressBook
6. sqlite3 AddressBook.sqlitedb
7. .tables
8. select * from ABPserson;
9. /private/var/wireless/Library/CallHistory
10. sqlite3 call_history.db
11. plutil -convert xml1 System/Library/Backup/Domains.plist
