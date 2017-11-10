---
author: null
categories:
- IOS
date: '2015-07-28'
description: null
photos: null
tags:
- Debug
title: xcode condition breakpoint non-ascii
---

#### 设置变量
```
expr username = @"username"
expr password = @"badpassword"
```

#### 条件断点
```
(BOOL)[item isEqualToString:@"three"]

//非 ASCII 码写法
(BOOL)[(NSString*)titleName isEqualToString:[NSString stringWithUTF8String:"消息"]]
```