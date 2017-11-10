---
author: null
categories:
- IOS
date: '2015-08-04'
description: null
photos: null
tags:
- IOS
title: dSYM parse tool
---

## dSYM 文件

### 什么是 dSYM 文件
Xcode编译项目后，我们会看到一个同名的 `dSYM` `文件，dSYM` 是保存 16 进制函数地址映射信息的中转文件，我们调试的 symbols 都会包含在这个文件中，并且每次编译项目的时候都会生成一个新的 dSYM 文件，位于 `/Users/<用户名>/Library/Developer/Xcode/Archives` 目录下，对于每一个发布版本我们都很有必要保存对应的 `Archives` 文件 ( AUTOMATICALLY SAVE THE DSYM FILES 这篇文章介绍了通过脚本每次编译后都自动保存 dSYM 文件)。


### dSYM 文件有什么作用
当我们软件 release 模式打包或上线后，不会像我们在 Xcode 中那样直观的看到用崩溃的错误，这个时候我们就需要分析 crash report 文件了，iOS 设备中会有日志文件保存我们每个应用出错的函数内存地址，通过 Xcode 的 Organizer 可以将 iOS 设备中的 DeviceLog 导出成 crash 文件，这个时候我们就可以通过出错的函数地址去查询 dSYM 文件中程序对应的函数名和文件名。大前提是我们需要有软件版本对应的 dSYM 文件，这也是为什么我们很有必要保存每个发布版本的 Archives 文件了。

<!--more-->

### 如何将文件一一对应
每一个 xx.app 和 xx.app.dSYM 文件都有对应的 UUID，crash 文件也有自己的 UUID，只要这三个文件的 UUID 一致，我们就可以通过他们解析出正确的错误函数信息了。

1) 查看 xx.app 文件的 UUID，terminal 中输入命令 ：
```
dwarfdump --uuid xx.app/xx (xx代表你的项目名)
```

2) 查看 xx.app.dSYM 文件的 UUID ，在 terminal 中输入命令：
```
dwarfdump --uuid xx.app.dSYM 
```

3) crash 文件内第一行 Incident Identifier 就是该 crash 文件的 UUID。

### dSYM工具
于是我抽了几个小时的时间将这些命令封装到一个应用中，也为以后解决bug提供了便利。

使用步骤:

1. 将打包发布软件时的xcarchive文件拖入软件窗口内的任意位置(支持多个文件同时拖入，注意：文件名不要包含空格)
2. 选中任意一个版本的xcarchive文件，右边会列出该xcarchive文件支持的CPU类型，选中错误对应的CPU类型。
3. 对比错误给出的UUID和工具界面中给出的UUID是否一致。
4. 将错误地址输入工具的文本框中，点击分析。


![](http://bcs.duapp.com/answerhuang/blog/dsymTool.png)

[Mac App](http://pan.baidu.com/s/1bnkxPvT)
[项目源码](https://github.com/answer-huang/dSYMTools)




















