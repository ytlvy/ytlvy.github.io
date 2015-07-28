title: xcode condition breakpoint non-ascii
categories: IOS
tags:
  - Debug
date: 2015-07-28 21:26:44
author:
description:
photos:
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
