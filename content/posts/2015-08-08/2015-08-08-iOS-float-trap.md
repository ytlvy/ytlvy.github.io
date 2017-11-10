---
author: null
categories:
- IOS
date: '2015-08-08'
description: null
photos: null
tags:
- Trap
title: iOS float trap
---

## float陷阱以及在Objc中的注意

浮点数不准，这个貌似基本都知道。但是在开发中很多人没有对它的使用产生警觉。如果你在开发Cocoa应用，你可能使用过如下代码判断系统版本：
```
if ([[[UIDevice currentDevice] systemVersion] floatValue] >= 7.0) {
  //something support for ios7
}
```

这样一段代码也的确工作良好，但是注意了如果你把比较的数值改为7.1，那么很有可能就会出问题。

### 浮点数不准
这一篇博文从二进制的表示方面说明了为什么浮点数不准，讲的很好，感兴趣的可以学习学习，学习这些感觉还是很有用的，也可以帮助我们定位一些诡异的问题。 [浮点数的二进制表示](http://www.ruanyifeng.com/blog/2010/06/ieee_floating-point_representation.html)
在精确的比较中不要使用float
文章开头的系统版本的比较就是一个典型的例子，我们可以通过其他方法实现比较：
```
if ([[[UIDevice currentDevice] systemVersion] compare:@"7.1"] != NSOrderedAscending) {
  //something support for ios7
}
```
