---
author: null
categories:
- IOS
date: '2015-07-17'
description: null
photos: null
tags:
- dig
- arc
title: ios arc dig
---

[转自](http://blog.sunnyxx.com/2014/03/15/objc_arc_secret/)

## objc arc的简单探索

### ARC or not？

`Automatic Reference Counting`是objc发展以来相当重要的一个进步

> 对于开发者，任何能降低开发难度，简化代码的功能，我们都应该去了解和使用。
我们应该利用一切“偷懒”的机会，将软件开发的复杂度分解并控制在一个个小的范围内，使得对于分解后的每一个小的任务，都能被新手掌握和维护。

基于简化开发的思想来看，ARC绝对是一个__没理由拒绝__的技术进步。
ARC随着iOS5问世，到现在iOS8都快出了，你还在手动写retain，release么？除了固守思想外，对ARC的恐惧大都来自对它的未知。

比如我在公司尝试说服team使用ARC时被质疑的几个问题：

1. ARC和Java的GC一样，会导致一部分性能损耗？
首先，ARC和GC是两码事，ARC是编译时编译器“帮你”插入了原本需要自己手写的内存管理代码，而非像GC一样运行时的垃圾回收系统

2. ARC内存不知道什么时候释放，导致不可控的内存涨落？
了解ARC的原理后，就知道，ARC下编译器插入的内存管理的代码是经过优化的，对于使用完的内存，多运行一行代码都不会浪费，可以这么说，手写的内存管理必须达到很严谨的水平才可能达到ARC自动生成的一样完整且没有疏漏

3. ARC下面自己不管理内存，很不爽，很没有安全感
这纯粹是习惯的问题了，开发者的目标是用最简化的手段完成一个最可靠的程序，进步需要改变的。好在编译选项中提供了`-fobjc-arc`和`-fno-objc-arc`来保证整个的变革的继续下去，就像社会主义中国里的港澳

### ARC的约定
使用ARC之后一个费解的地方是，一个方法生成的对象，没有任何附加标示，ARC怎么知道生成的对象是不是`autorelease`的呢？

```
@interface Sark : NSObject
+ (instancetype)sarkWithMark:(NSString *)mark; // 1
- (instancetype)initWithMark:(NSString *)mark; // 2
@end
```

这是非ARC时常用的手段，1生成autorelease对象，2生成普通对象，而现在ARC不能调用autorelease，使用时怎么能知道呢？

```
{
    // ...
    Sark *sark1 = [Sark sarkWithMark:@"萨萨萨"];
    Sark *sark2 = [[Sark alloc] initWithMark:@"萨萨萨"];
}
```

使用约定，NS定义了下面三个编译属性

```
#define NS_RETURNS_RETAINED __attribute__((ns_returns_retained))
#define NS_RETURNS_NOT_RETAINED __attribute__((ns_returns_not_retained))
#define NS_RETURNS_INNER_POINTER __attribute__((objc_returns_inner_pointer))
```

这三个属性是Clang自己使用的标示，除非__特殊情况__不要自己使用，但是这些对理解ARC是很有帮助的。
这里还要介绍一个概念，`Method family`

> An Objective-C method may fall into a method family, which is a conventional set of behaviors ascribed to it by the Cocoa conventions.

指的是命名上表示一类型的方法，比如`- init`和`- initWithMark:`都属于init的family
于是乎，编译器约定，对于`alloc`, `init`, `copy`, `mutableCopy`, `new`这几个家族的方法，后面默认加`NS_RETURNS_RETAINED`标识；而其他不指名标识的family的方法默认添加`NS_RETURNS_NOT_RETAINED`标识
也就是说刚才的方法，在编译器看来是这样的：

```
@interface Sark : NSObject
+ (instancetype)sarkWithMark:(NSString *)mark NS_RETURNS_NOT_RETAINED; // 1
- (instancetype)initWithMark:(NSString *)mark NS_RETURNS_RETAINED; // 2
@end
```

这也就是为什么ARC下面，不能把一个属性定义成名字是这样的：

```
@property (nonatomic, copy) NSString *newString; // 编译器不允许
```

`- newString`就成了new家族的方法，内存就不对了
对于`NS_RETURNS_INNER_POINTER`这货，主要使用在返回的是一个对象的内部C指针的情况，如NSString的方法：

```
- (__strong const char *)UTF8String NS_RETURNS_INNER_POINTER;
```

就使用了这个标识，这个就不深入研究了，直接上文档：

> An Objective-C method returning a non-retainable pointer may be annotated with the objc_returns_inner_pointer attribute to indicate that it returns a handle to the internal data of an object, and that this reference will be invalidated if the object is destroyed. When such a message is sent to an object, the object’s lifetime will be extended until at least the earliest of:
the last use of the returned pointer, or any pointer derived from it, in the calling function or
the autorelease pool is restored to a previous state.







