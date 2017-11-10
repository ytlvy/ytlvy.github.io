---
author: null
categories:
- IOS
date: '2015-08-04'
description: null
photos: null
tags:
- Debug
title: parse crashLog manually
---

[转自](http://foggry.com/blog/2015/07/27/ru-he-shou-dong-jie-xi-crashlog/)

## 如何手动解析CrashLog

解决崩溃问题是移动应用开发者最日常的工作之一。如果是开发过程中遇到的崩溃，可以根据重现步骤调试，但线上版本就无能为力了。好在目前已经有很多不错的第三方CrashLog搜集平台（如友盟、Crashlytics等）为我们做好了解析工作，甚至在Xcode7里苹果也跟进了解析线上版本崩溃日志的功能，为开发者减轻了不少负担。尽管通常已经不需要我们手工处理CrashLog，了解CrashLog的还原原理和方法还是有必要的。

### .dSYM
`.dSYM(debugging SYMbols)`又称为调试符号表，是苹果为了方便调试和定位问题而使用的一种调试方案，本质上使用的是起源于贝尔实验室的DWARF（Debugging With Attributed Record Formats），其在`.xcarchive`目录中的层次结构为：
```
.xcarchive
--dSYMs
  |--Your.app.dSYM
    |--Contents
      |--Resources
        |--DWARF
```

关于DWARF的具体内容以后有机会再说。我们能解析CrashLog全靠.dSYM文件，解析方式见后文。

<!--more-->

### 确定符号表和崩溃日志的一致性
有了符号表文件，有了崩溃日志文件，在解析之前一定要确保二者的对应关系，否则就算按照下述步骤解析出内容也肯定是不准确的。二者的对应关系可以通过UUID来确定。

### 从崩溃日志中获取UUID
崩溃日志比较靠下的位置有个`Binary Images`模块，其第一行内容如下：

```
Binary Images:
0xa2000 - 0x541fff Your armv7  <a5c8d3cfda65396689e4370bf3a0ac64> /var/mobile/Containers/Bundle/Application/645D3184-4C20-4161-924B-BDE170FA64CC/Your.app/Your
```

从中可以看到关于你应用的若干信息：

- 代码段的起终地址为：0xa2000 – 0x541fff
- 运行你应用的CPU指令集为：armv7
- 应用的UUID为：a5c8d3cfda65396689e4370bf3a0ac64（不区分大小写）

#### 从符号表中获取UUID
执行以下命令从符号表中提取UUID：
```
dwarfdump --uuid Your.app.dSYM
```

或者：
```
dwarfdump --uuid Your.app.dSYM/Contents/Resources/DWARF/Your
```

执行结果为：
```
UUID: A5C8D3CF-DA65-3966-89E4-370BF3A0AC64 (armv7) Your.app.dSYM/Contents/Resources/DWARF/Your
```

由此得到armv7指令集的UUID为：A5C8D3CF-DA65-3966-89E4-370BF3A0AC64（如果你的二进制文件支持多个指令集，这里会列出每个指令集对应符号表的UUID），通过和崩溃日志中的对比发现二者一致，才可进行进一步的解析操作。

### 计算崩溃符号表地址
以下面的崩溃堆栈为例：
```
Thread 0:
0   libobjc.A.dylib                   0x33f10f60 0x33efe000 + 77664
1   Foundation                        0x273526ac 0x2734a000 + 34476
2   Foundation                        0x27355c3e 0x2734a000 + 48190
3   UIKit                             0x29ef9d1c 0x29bbc000 + 3398940
4   UIKit                             0x29ef9c9a 0x29bbc000 + 3398810
5   UIKit                             0x29ef954c 0x29bbc000 + 3396940
6   UIKit                             0x29c3a16a 0x29bbc000 + 516458
7   UIKit                             0x29e4b8e6 0x29bbc000 + 2685158
8   UIKit                             0x29c3a128 0x29bbc000 + 516392
9   Your                              0x000f0846 0xa2000 + 321606
10  UIKit                             0x29e90fb2 0x29bbc000 + 2969522
11  UIKit                             0x29e91076 0x29bbc000 + 2969718
12  UIKit                             0x29e867cc 0x29bbc000 + 2926540
13  UIKit                             0x29c9e8ea 0x29bbc000 + 927978
14  UIKit                             0x29bc8a6a 0x29bbc000 + 51818
15  QuartzCore                        0x295f0a08 0x295e4000 + 51720
16  QuartzCore                        0x295ec3e0 0x295e4000 + 33760
17  QuartzCore                        0x295ec268 0x295e4000 + 33384
18  QuartzCore                        0x295ebc4c 0x295e4000 + 31820
19  QuartzCore                        0x295eba50 0x295e4000 + 31312
20  QuartzCore                        0x295e5928 0x295e4000 + 6440
21  CoreFoundation                    0x266d0d92 0x26604000 + 839058
22  CoreFoundation                    0x266ce44e 0x26604000 + 828494
23  CoreFoundation                    0x266ce856 0x26604000 + 829526
24  CoreFoundation                    0x2661c3bc 0x26604000 + 99260
25  CoreFoundation                    0x2661c1ce 0x26604000 + 98766
26  GraphicsServices                  0x2da1a0a4 0x2da11000 + 37028
27  UIKit                             0x29c2a7ac 0x29bbc000 + 452524
28  Your                              0x0024643a 0xa2000 + 1721402
29  libdyld.dylib                     0x34484aac 0x34483000 + 6828
```

#### 1、 符号表堆栈地址计算方式
要想利用符号表解析出崩溃对应位置，需要计算出符号表中对应的崩溃堆栈地址。而从上述堆栈中第9行可以看到，应用崩溃发生在运行时地址`0x000f0846`，该进程的运行时起始地址是`0xa2000`，崩溃处距离进程起始地址的偏移量为十进制的`321606`(对应十六进制为`0x4E846`)。三者对应关系：
```
0x000f0846 = 0xa2000 + 0x4E846
```

对应的公式为：
```
运行时堆栈地址 = 运行时起始地址 + 偏移量
```

崩溃堆栈中的起始地址和崩溃地址均为运行时地址，根据虚拟内存偏移量不变原理，只要提供了符号表TEXT段的起始地址，再加上偏移量（这里为0x4E846）就能得到符号表中的堆栈地址，即：
```
符号表堆栈地址 = 符号表起始地址 + 偏移量
```

#### 2、获取符号表中的TEXT段起始地址

符号表中TEXT段的起始地址可以通过以下命令获得：
```
$otool -l Your.app.dSYM/Contents/Resources/DWARF/Your
```

运行结果中的片段如下：
```
Load command 3
      cmd LC_SEGMENT
  cmdsize 736
  segname __TEXT
   vmaddr 0x00004000
   vmsize 0x00700000
  fileoff 0
 filesize 0
  maxprot 0x00000005
 initprot 0x00000005
   nsects 10
    flags 0x0
```

其中的`vmaddr 0x00004000`字段即为TEXT段的起始地址。

#### 3、计算符号表地址
由公式：

```
符号表堆栈地址 = 符号表起始地址 + 偏移量
```

可得：
```
0x52846 = 0x4E846 + 0x4000
```

即符号表中的崩溃地址为`0x52846`，接下来就可以根据这个地址解析出崩溃位置了。

### 崩溃信息还原
有了符号表的崩溃地址，有以下几种方式解析崩溃信息：

#### dwarfdump
命令如下：
```
$dwarfdump --arch armv7 Your.app.dSYM --lookup 0x52846 | grep 'Line table'
```

需要注意的是：

- 这里的armv7是运行设备的CPU指令集，而不是二进制文件的指令集, 比如armv7指令集的二进制文件运行在arm64指令集的设备上，这个地方应该写arm64。

- `—lookup`后面跟的一定是经过准确计算的符号表中的崩溃地址
- 使用dwarfdump解析的结果较杂乱，因此使用grep命令抓取其中关键点展示出来


运行结果如下：
```
Line table dir : '/data/.../Src/OBDConnectSetting/Controller'
Line table file: 'OBDFirstConnectViewController.m' line 882, column 5 with start address 0x000000000052768
```
其中第一行是编译时文件目录，第二行包含了崩溃发生的文件名称以及文件中具体行号等信息，有了这些信息就能准确定位崩溃原因啦。

#### atos
atos是另一种更加简洁的崩溃日志解析方法，使用方式如下：
```
 $atos -o LuBao -arch armv7 0x52846
```

其执行结果如下：
```
-[OBDFirstConnectViewController showOilPricePickerView] (in Your) (OBDFirstConnectViewController.m:882)
```

相对`dwarfdump`命令的解析结果，更加简洁直观的指出了崩溃发生的位置。

#### 无需符号表崩溃地址的解析方式
实际上，`atos`还提供了另外一种无需计算崩溃地址对应的符号表地址的方式，命令格式如下：

```
$atos -o Your.app.dSYM/Contents/Resources/DWARF/Your -arch armv7 -l 0xa2000 0x000f0846
```

其中-l选项指定了二进制文件在运行时的起始地址0xa2000（获取方式见Binary Images相关内容）,后面跟的是崩溃发生的运行时地址0x000f0846，解析结果和使用计算得到的符号表中崩溃地址一致：
```
-[OBDFirstConnectViewController showOilPricePickerView] (in Your) (OBDFirstConnectViewController.m:882)
```

### 参考文档

[How to Match a Crash Report to a Build](https://developer.apple.com/library/mac/qa/qa1765/_index.html)
[CrashReporter](https://developer.apple.com/library/mac/technotes/tn2004/tn2123.html)
[Understanding and Analyzing iOS Application Crash Reports](https://developer.apple.com/library/ios/technotes/tn2151/_index.html)
[atos and dwarfdump won’t symbolicate my address](http://stackoverflow.com/a/12464678/2293677)



