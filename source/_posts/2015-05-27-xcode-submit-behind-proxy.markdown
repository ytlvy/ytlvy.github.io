---
layout: post
title: "代理环境下 xcode submit 失败解决---xcode6.3更新"
date: 2015-05-27 15:51:11 +0800
comments: true
categories: Xcode
---
由于公司网络需要设置代理才能访问外网，此时通过xocde提交应用到itunesconnect的时候会出现443访问失败的错误，可以通过下面的方法来解决。

>1. Go to Application Loader java folder : 
>/Applications/Xcode.app/Contents/Applications/"Application Loader.app"/Contents/MacOS/itms/java/lib
>
>>//xcode6.3
>>/Applications/Xcode.app/Contents/Applications/"Application Loader.app"/Contents/itms/java/lib

>2. Open net.properties file with "sublime text" or "text mate"

> 3. Change "#https.proxyPort=443" proxy port to "https.proxyPort=80"

> 4. Save the file and reopen Application Loader and Try again.