---
layout: post
title: "IOS8.3 NSMutableURLRequest需手动设置Content-Type"
date: 2015-05-27 15:54:55 +0800
comments: true
categories: IOS
---
引用：http://stackoverflow.com/questions/29528293/nsmutableurlrequest-body-malformed-after-ios-8-3-update

IOS8.3开始 NSMutableURLRequest需要手动设置Content-Type，否则导致服务器接受参数失败的错误，请添加如下代码:

```
[request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
```