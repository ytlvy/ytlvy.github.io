title: iOS static library architectures
categories: IOS
tags:
  - Static Library
  - x86_64
date: 2015-07-26 08:12:28
author:
description:
photos:
---

## iOS static library architectures

最近入职, 刚接手需要参与的工程, 调试了半天, 发现由于32\64位的问题, 不能在模拟器调试, 很郁闷. 后来google 到答案记录如下:

静态库编译支持`arm64,armv7 armv7s, i386, x86_64`.

1) 分别在设备和模拟器编译
2) 找到`Build/Products`目录, 下有两个目录 `Release-iphoneos`  `Release-iphonesimulator`

```
/Users/kappe/Library/Developer/Xcode/DerivedData/zbar-gyozyrpbqzvslmfoadhqkwskcesd/Build/Products
```

3) 合并
```
lipo -create Release-iphoneos/libzbar.a Release-iphonesimulator/libzbar.a -o libzbar.a
```
