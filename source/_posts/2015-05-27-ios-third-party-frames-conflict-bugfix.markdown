---
layout: post
title: "第三方类库重复冲突解决"
date: 2015-05-27 15:54:42 +0800
comments: true
categories: IOS
---
## 第三方类库重复冲突解决
### 复制.a文件
```
cd ~/ && mkdir librepack && cd librepack
cp /Users/tony/Desktop/XXXProject/Lib/libMiPushSDK.a ./libx.a
```

### 查看包信息
`lipo -info libx.a`

### 解包
如果提示fat file，那么代表这个包是支持多平台的，例如armv7，armv7s，i386等，这需要我们逐一做解包重打包操作。否则我们只需要做一次[1-6]操作即可

1. 创建临时文件夹，用于存放armv7平台解压后的.o文件：`mkdir armv7`
2. 取出armv7平台的包：`lipo libx.a -thin armv7 -output armv7/libx-armv7.a`
3. 查看库中所包含的文件列表：`ar -t armv7/libx-armv7.a`
4. 解压出object file（即.o后缀文件）：`cd armv7 && ar xv libx-armv7.a`
5. 找到冲突的包（JSONKit），删除掉`rm JSONKit.o`
6. 重新打包object file：`cd .. && ar rcs libx-armv7.a armv7/*.o`，可以再次使用[2]中命令确认是否已成功将文件去除
7. 将其他几个平台(armv7s, i386)包逐一做上述[1-6]操作
8. 重新合并为fat file的.a文件：
`lipo -create libx-armv7.a libx-armv7s.a libx-i386.a -output libMiPushSDK-new.a`
9. 拷贝到项目中覆盖源文件：
`cp libMiPushSDK-new.a /Users/tony/Desktop/XXXProject/Lib/libMiPushSDK.a`

参考:http://itony.me/674.html