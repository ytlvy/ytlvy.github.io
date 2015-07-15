---
layout: post
title: "ios reverse"
date: 2015-05-30 22:26:18 +0800
comments: true
categories: [IOS, reverse]
---
### class-dump-z
#### 下载配置class_dump_z
前往 https://code.google.com/p/networkpx/wiki/class_dump_z ，下载tar包，然后解压配置到本地环境
```
    $ tar -zxvf class-dump-z_0.2a.tar.gz  
    $ sudo cp mac_x86/class-dump-z /usr/bin/  
```

#### class_dump 分文件
    class-dump-z App -H -o outputdir

#### class_dump支付宝app
    $ class-dump-z Portal > Portal-dump.txt  
    
    @protocol XXEncryptedProtocol_10764b0  
    -(?)XXEncryptedMethod_d109df;  
    -(?)XXEncryptedMethod_d109d3;  
    -(?)XXEncryptedMethod_d109c7;  
    -(?)XXEncryptedMethod_d109bf;  
    @optional  
    -(?)XXEncryptedMethod_d109eb;  
    @end  
查看得到的信息是加过密的，这个加密操作是苹果在部署到app store时做的，所以我们还需要做一步解密操作

<!--more-->

### 使用Clutch解密支付宝app
#### iOS7越狱后的Cydia源里已经下载不到Clutch了，但是我们可以从网上下载好推进iPhone
地址:https://github.com/KJCracks/Clutch-dl/releases

#### 查看可解密的应用列表
    root# ./Clutch   
    
    Clutch-1.3.2  
    usage: ./Clutch [flags] [application name] [...]  
    Applications available: 9P_RetinaWallpapers breadtrip Chiizu CodecademyiPhone FisheyeFree food GirlsCamera IMDb InstaDaily InstaTextFree iOne ItsMe3 linecamera Moldiv MPCamera MYXJ NewsBoard Photo Blur Photo Editor PhotoWonder POCO相机 Portal QQPicShow smashbandits Spark tripcamera Tuding_vITC_01 wantu WaterMarkCamera WeiBo Weibo


#### 解密支付宝app
    root# ./Clutch Portal

### 导出已解密的支付宝app
从上一步骤得知，已解密的ipa位置为：/var/root/Documents/Cracked/支付宝钱包-v8.0.0-(Clutch-1.3.2).ipa
将其拷贝到本地去分析

### class_dump已解密的支付宝app
解压.ipa后，到 支付宝钱包-v8.0.0-(Clutch-1.3.2)/Payload/Portal.app 目录下，class_dump已解密的二进制文件

    class-dump-z Portal > ~/Portal-classdump.txt


### iOS运行时工具-cycript
网址：http://www.cycript.org/


### 使用introspy追踪分析应用程序
1. 下载 https://github.com/iSECPartners/Introspy-iOS/releases 将其拷贝到设备中，并执行安装命令
`# dpkg -i com.isecpartners.introspy-v0.4-ios_7.deb `

2. 重启设备：
   `# killall SpringBoard  `
3. 到设置中，就可以查看到instrospy的设置选项了
4. 在Introspy-Apps中选择要跟踪的app名称。Instrospy-Settings则提供一些常规跟踪设置选项，默认是全部开启
5. 然后启动想要跟踪的应用程序，就可以直接查看log获取Instrospy为我们跟踪捕获的信息，
这里以跟踪支付宝app为例
6. 打开支付宝app，选择添加银行卡，随意添加一个卡号，然后点击下一步
7. 支付宝app反馈添加失败，该卡暂不支持，Instrospy捕获的信息也很清晰：
![](http://img.blog.csdn.net/20140208142548437?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveWl5YWFpeHVleGk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
8. 追踪信息被保存为一个数据库introspy-com.alipay.iphoneclient.db，存放在：
./private/var/mobile/Applications/4763A8A5-2E1D-4DC2-8376-6CB7A8B98728/Library/
introspy-com.alipay.iphoneclient.db
9. 也可以借助Introspy-Analyzer在本地将该数据库解析成一个直观的report.html查看
10. 将introspy-com.alipay.iphoneclient.db拷贝到本地，执行：
python introspy.py -p ios --outdir Portal-introspy-html introspy-com.alipay.iphoneclient.db  
11. 就会生成一个 Portal-introspy-html  文件夹，该目录下有 report.html ，用浏览器打开
    `open report.html`
12. 就可以清晰的查看追踪信息了，主要分为DataStorage、IPC、Misc、Network、Crypto六大类信息
举个例子，选择Crypto可以查看支付宝app采取了什么加密措施，如果你看过我之前的文章，
一定会一眼就认出来手势密码的：


### 使用Cycript修改支付宝app运行时
#### 安装Cycript
到[Cycript](http://www.cycript.org/)官方网站下载资源工具，然后推进已越狱的iPhone中，进行安装：

    dpkg -i cycript_0.9.461_iphoneos-arm.deb  
    dpkg -i libffi_1-3.0.10-5_iphoneos-arm.deb  

#### 确定支付宝进程
运行支付宝app，然后获取它的进程号：

    Primer:/ root# ps aux | grep Portal 

#### Cycript钩住支付宝进程
    cycript -p 479  

#### 获取当前界面的viewController并修改背景色
```bash
cy# var app = [UIApplication sharedApplication]  
@"<DFApplication: 0x16530660>"  
  
cy# app.delegate  
@"<DFClientDelegate: 0x165384d0>"  
  
cy# var keyWindow = app.keyWindow  
@"<UIWindow: 0x1654abb0; frame = (0 0; 320 568); gestureRecognizers = <NSArray: 0x1654b190>; layer = <UIWindowLayer: 0x1654ace0>>"  
  
cy# var rootController = keyWindow.rootViewController  
@"<DFNavigationController: 0x1654b6c0>"  
  
cy# var visibleController = rootController.visibleViewController  
@"<ALPLauncherController: 0x166acfb0>"  
  
cy# visibleController.childViewControllers  
@["<HPHomeWidgetGroup: 0x166afbc0>","<ADWRootViewController: 0x165745c0>","<ALPAssetsRootViewController: 0x16577250>","<SWSecurityWidgetGroup: 0x166bd940>"]  
  
cy# var assetsController = new Instance(0x16577250)  
@"<ALPAssetsRootViewController: 0x16577250>"  
  
cy# assetsController.view.backgroundColor = [UIColor blueColor]  
@"UIDeviceRGBColorSpace 0 0 1 1"  
```




### gbd
> Cycript can not set break point
> Cycript can not alter value of variable and register

```bash
//attach GDB-Demo.PID
(gdb) attach GDB-Demo.1438

(gdb)break loginButtonTapped:
(gdb)c
(gdb)disas                  //print disassemble of the function
//whenever an external method is called or a property is accessed, the objc_msgSend function is called
// blx      ===== objc_msgSend
(gdb)break *0x0007e428
(gdb)b *0x0007e45e
(gdb)c
(gdb) x/s $r1
(gdb) po $r2        //Admin
(gdb)x/a $2         //0x80910  __CFConstantStringClassReference
(gdb)po    0x80910

//break cmp address
(gdb)b *0x0007e4a8
(gdb)b *0x0007e52c
(gdb)c
(gdb)set $r0 = 1
(gdb)set $r0 = 1
(gdb)c


```

```bash
    break objc_msgSend
    c                    //continue
    help x               // 显示命令
    x/a $r0              //object  show  register 0
    x/s $r1              //string
    x/s $r2              //
```

#### 批处理命令
```bash
    (gdb)commands
    >x/a  $r0
    >x/s  $r1
    >c
    >end
    c
```

```bash
    (gdb)commands
    >printf "[%s, %s]\n",(char *)class_getName(**$r0),$1
    >c
    >end

```

### JailBroken bypass
1. class-dump-z find isJailBroken;
2. Cycript -p 2166
3. JailbreakDetector->isa.messages
4. JailbreakDetector->isa.messages['isJailbroken'] = function(){return false;}

###Ida patch
~/Library/Application\ Support/iPhone\ Simulator/
~/Library/Developer/CoreSimulator/Devices/<device-specific-hash>/data/Container‌​s/Bundle/Application/<app-specific-hash>


#### IDA
1. show graphical format, double click on the function name in the functions window 
and press Space

2. If you want to switch to the previous view, you can press Space again
3.  As we can clearly see, the address of this instruction is 00002CB1. However, we cannot just go ahead and change the address at this instruction, This is because this address is the absolute address of this instruction and it will be different every time the application is launched. What we need to find out is the offset of this instruction relative to the Mach-O binary. This instruction has to be modified in the code section of the assembly. Hence the offset of this instruction relative to the binary can be calculated as
```
//(Offset of code section relative to binary) +  (Absolute address of the instruction to be changed -  Starting address of the code section)
Absolute address : 00002A62

otool -l GDB-Demo
//Look for the text section
Section
  sectname __text
   segname __TEXT
      addr 0x00000001000016c9
      size 0x0000000000000329
    offset 5833
     align 2^0 (1)
    reloff 0
    nreloc 0
     flags 0x80000400
 reserved1 0
 reserved2 0


 start address: 0x00000001000016c9
 offset: 5833

 5833(Decimal) + (0x00002A62(Hex)- 0x00000001000016c9(Hex)) = 0x1cb1
```

### URL Schemes
```html
    <script>document.location=‘tel://1123456789’<script>
    <iframe src=“tel://1123456789”></iframe>
```

### IRET
```bash
wget --no-check-certificate https://www.veracode.com/sites/default/files/Resources/Tools/iRETTool.zip

unzip iRETTool.zip
dpkg -i iRET.deb  //reboot your device then launch the app

//install dumpdecrypted
```

### dumpdecrypted
1. download file
2. make
3. copy .dylib to your device//drag and drop inside the folder /Library using iExplorer

### Theos
```bash
export THEOS=/opt/theos
svn co http://svn.howett.net/svn/theos/trunk $THEOS

```

#### ldid
ldid的作用是模拟给iPhone签名的流程，使得你能够在真实的设备上安装越狱的apps/hacks

    cp ~/Copy/ios/ldid /opt/theos/bin/ldid

#### dkpg

    brew install dkpg

#### theos 使用
1. 下载libsubstrate.dylib，然后copy到/opt/theos/lib http://www.mediafire.com/download/2upm53uzzj0488u/libsubstrate.dylib
2．iOS 头文件
很可能theos本身就自带了你所需要的头文件，但是，如果你编译程序的时候提示你头文件相关的问题，那你就需要准备相关的头文件了。要么从设备上dump头文件，要么google，建议你先google一下，看其他人有没已经提供了这些头文件。
一旦你有这些头文件，记得把它们放在/opt/theos/include
3. 创建项目
`$THEOS/bin/nic.pl`
4. The Tweak File
```
vim Tweak.xm
%hook Springboard // overwrite methods here %end 

#import<SpringBoard/SpringBoard.h>  
%hook SpringBoard 
 -(void)applicationDidFinishLaunching:(id)application { 
%orig; 
 UIAlertView *alert = [[UIAlertViewalloc] 
initWithTitle:@"Welcome"  message:@"Welcome to your iPhone Brandon!"  
delegate:nilcancelButtonTitle:@"Thanks"  
otherButtonTitles:nil] 
[alert show]; 
[alert release];
 } 
 %end

```
5.  添加Framework
```
vim Makefile

WelcomeWagon_FRAMEWORKS = UIKit//add
```
6. Building, Packaging, Installing
```
make
make package
Dpkg –i     //install
dpkg -r     //remove
```


### logify
```bash
mkdir Headers
class-dump-z App -H -o Headers
cd Headers//check header files

//add trace the method 
/opt/theos/bin/logify.pl ClientSideInjectionVC.h > Tweak.xm
/opt/theos/bin/logify.pl JailbreakDetectionVC.h > Tweak.xm
/opt/theos/bin/logify.pl DamnVulnerableAppUtilities.h > Tweak.xm
 
```
