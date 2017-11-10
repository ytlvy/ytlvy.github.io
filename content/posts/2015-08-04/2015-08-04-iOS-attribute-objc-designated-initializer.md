---
author: null
categories:
- IOS
date: '2015-08-04'
description: null
photos: null
tags:
- attribute
title: iOS attribute objc_designated_initializer
---

[reference](https://leafduo.com/articles/2014/03/20/objc-designated-initializer/)

## __attribute__((objc_designated_initializer))
指定初始化器（`designated initializer`）是 Objective-C 中的一个重要的概念，但是很可惜的是，很多开发者（不知道为什么）并没能正确遵守关于指定初始化器的一些惯例。 

之前，我们只能通过 code review 之类的方法来找出、修正这些问题；现在，clang 为我们提供了编译器级别的支持，能够找出不遵守管理的地方并给出警告，我们需要做的是标记出哪个初始化器是指定初始化器，例如：
```
@interface FTBObject

- (instancetype)init __attribute__((objc_designated_initializer));

@end
```

通过在接口定义中指定指定初始化器，不仅编译器能够给出相关的警告，而且需要继承该类的开发者也可以不参考文档就知道哪个初始化器是指定初始化器。

目前，clang 可以提供一系列警告，例如非指定初始化器没有调用其他初始化器、指定初始化器没有调用父类的指定初始化器、指定初始化器调用了非指定初始化器、子类没有复写父类的指定初始化器等。 具体的例子可以看这个[测试用例](https://llvm.org/svn/llvm-project/cfe/trunk/test/SemaObjC/attr-designated-init.m)。

不幸的是，目前系统库的接口还没有这个属性，也就是说只有继承自己的代码和标记了这个指令的第三方库中的类才有用。















