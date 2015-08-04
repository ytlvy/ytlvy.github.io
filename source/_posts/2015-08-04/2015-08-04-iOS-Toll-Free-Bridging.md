title: iOS Toll-Free Bridging
categories: IOS
tags:
  - Memory
date: 2015-08-04 21:28:29
author:
description:
photos:
---
[reference](http://gracelancy.com/blog/2014/04/21/toll-free-bridging/)

## Toll-Free Bridging

### 什么是 Toll-Free Bridging
有一些数据类型是能够在 `Core Foundation Framework` 和 `Foundation Framework` 之间交换使用的。这意味着，对于同一个数据类型，你既可以将其作为参数传入 `Core Foundation`函数，也可以将其作为接收者对其发送 `Objective-C` 消息（即调用ObjC类方法）。这种在 `Core Foundation 和 Foundation` 之间交换使用数据类型的技术就叫 `Toll-Free Bridging`.

举例说明，`NSString`和`CFStringRef`即是一对可以相互转换的数据类型：

```
// ARC 环境下
// Bridging from ObjC to CF
NSString *hello = @"world";
CFStringRef world = (__bridge CFStringRef)(hello);
NSLog(@"%ld", CFStringGetLength(world));

// Bridging from CF to ObjC
CFStringRef hello = CFStringCreateWithCString(kCFAllocatorDefault, "hello", kCFStringEncodingUTF8);
NSString *world = (__bridge NSString *)(hello);
NSLog(@"%ld", world.length);
CFRelease(hello);
```

大部分（但不是所有！）`Core Foundation` 和 `Foundation` 的数据类型可以使用这个技术相互转换，Apple 的文档里有一个列表（[传送门](https://developer.apple.com/library/ios/documentation/General/Conceptual/CocoaEncyclopedia/Toll-FreeBridgin/Toll-FreeBridgin.html)），列出了支持这项技术的数据类型。
MRC 下的 `Toll-Free Bridging` 因为不涉及内存管理的转移，可以直接相互 `bridge` 而不必使用类似`__bridge`修饰字，我们之后再讨论这个问题。


### Toll-Free Bridging 是如何实现的？

1) 每一个能够 bridge 的 ObjC 类，都是一个类簇（`class cluster`）。类簇是一个公开的抽象类，但其核心功能的是在不同的私有子类中实现的，公开类只暴露一致的接口和实现一些辅助的创建方法。而与该 ObjC 类相对应的 `Core Foundation` 类的内存结构，正好与类簇的其中一个私有子类相同。

举个例子，NSString是一个类簇，一个公开的抽象类，但每次创建一个NSString的实例时，实际上我们会获得其中一个私有子类的实例。而NSString的其中一个私有子类实现既为NSCFString，其内存的结构与CFString是相同的，CFString的isa指针就指向NSCFString类，即，CFString对象就是一个NSCFString类的实例。

所以，当`NSString`的实现刚好是`NSCFString`的时候，他们两者之间的转换是相当容易而直接的，他们就是同一个类的实例。

2) 当`NSString`的实现不是`NSCFString`的时候（比如我们自己 subclass 了`NSString`），我们调用 CF 函数，就需要先检查对象的具体实现。如果发现其不是`NSCFString`，我们不会调用 CF 函数的实现来获得结果，而是通过给对象发送与函数功能相对应的 ObjC 消息（调用相对应的`NSString`的接口）来获得其结果。
例如`CFStringGetLength`函数，当收到一个作为参数传递进来的对象时，会先确认该对象到底是不是`NSCFString`实现。如果是的话，就会直接调用`CFStringGetLength`函数的实现来获得字符串的长度；如果不是的话，会给对象发送length消息（调用`NSString`的`- (NSUInteger)length`接口），来得到字符串的长度。
通过这样的技术，即使是我们自己子类了一个`NSString`，也可以和`CFStringRef`相互 Bridge。

3) 其他支持 Toll-Free Bridging 的数据类型原理也同NSString一样，比如NSNumber的NSCFNumber和CFNumber。

### ARC 下的 Toll-Free Bridging
如之前提到的，MRC 下的 `Toll－Free Bridging` 因为不涉及内存管理的转移，相互之间可以直接交换使用：
```
// bridge
NSString *nsStr = (NSString *)cfStr;
CFStringRef cfStr = (CFStringRef)nsStr;

// 调用函数或者方法
NSUInteger length = [(NSString *)cfStr length];
NSUInteger length = CFStringGetLength((CFStringRef)nsStr);

// release
CFRelease((CFStringRef)nsStr);
[(NSString *)cfStr release];
```

而在 ARC 下，事情就会变得复杂一些，因为 ARC 能够管理 Objective-C 对象的内存，却不能管理 CF 对象，CF 对象依然需要我们手动管理内存。在 CF 和 ObjC 之间 bridge 对象的时候，问题就出现了，编译器不知道该如何处理这个同时有 ObjC 指针和 CFTypeRef 指向的对象。

这时候，我们需要使用`__bridge`, `__bridge_retained`, `__bridge_transfer` 修饰符来告诉编译器该如何去做。

### __bridge
最常用的修饰符，这意味着告诉编译器不做任何内存管理的事情，编译器仍然负责管理好在 Objc 一端的引用计数的事情，开发者也继续负责管理好在 CF 一端的事情。举例说明：

例子1:
```
// objc to cf
NSString *nsStr = [self createSomeNSString];
CFStringRef cfStr = (__bridge CFStringRef)nsStr;
CFUseCFString(cfStr);
// CFRelease(cfStr); 不需要
```

在这里，编译器会继续负责nsStr的内存管理的事情，不会在 bridge 的时候 retain 对象，所以也不需要开发者在 CF 一端释放。需要注意的是，当nsStr被释放的时候（比如出了作用域），意味着cfStr指向的对象被释放了，这时如果继续使用cfStr将会引起程序崩溃。

例子2:
```
// cf to objc
CFStringRef hello = CFStringCreateWithCString(kCFAllocatorDefault, "hello", kCFStringEncodingUTF8);
NSString *world = (__bridge NSString *)(hello);
CFRelease(hello); // 需要
[self useNSString:world];
```

在这里，bridge 的时候编译器不会做任何内存管理的事情，bridge 之后，会负责 ObjC 一端的内存管理的事情 。同时，开发者需要负责管理 CF 一端的内存管理的事情，需要再 bridge 之后，负责 release 对象。

### __bridge_retained
接`__bridge`一节的第一个例子，objc to cf。为了防止nsStr被释放，引起我们使用cfStr的时候程序崩溃，可以使用`__bridge_retained`修饰符。这意味着，在 bridge 的时候，编译器会 retain 对象，而由开发者在 CF 一端负责 release。这样，就算nsStr在 objc 一端被释放，只要开发者不手动去释放cfStr，其指向的对象就不会被真的销毁。但同时，开发者也必须保证和负责对象的释放。例如：
```
// objc to cf
NSString *nsStr = [self createSomeNSString];
CFStringRef cfStr = (__bridge_retained CFStringRef)nsStr;
CFUseCFString(cfStr);
CFRelease(cfStr); // 需要
```

### __bridge_transfer
接`__bridge`一节的第二个例子，cf to objc。我们发现如果使用__bridge修饰符在cf转objc的时候非常的麻烦，我们既需要一个CFTypeRef的变量，还需要在 bridge 之后负责释放。这时我们可以使用`__bridge_transfer`，意味着在 bridge 的时候，编译器转移了对象的所有权，开发者不再需要负责对象的释放。例如：
```
// cf to objc
CFStringRef hello = CFStringCreateWithCString(kCFAllocatorDefault, "hello", kCFStringEncodingUTF8);
NSString *world = (__bridge_transfer NSString *)(hello);
// CFRelease(hello); 不需要
[self useNSString:world];
```

甚至可以这么写：
```
// cf to objc
NSString *world = (__bridge_transfer NSString *)CFStringCreateWithCString(kCFAllocatorDefault, "hello", kCFStringEncodingUTF8);
[self useNSString:world];
```

## 小结

- `(__bridge T) op`：告诉编译器在 bridge 的时候不要做任何事情
- `(__bridge_retained T) op`：（ ObjC 转 CF 的时候使用）告诉编译器在 bridge 的时候 retain 对象，开发者需要在CF一端负责释放对象
- `(__bridge_transfer T) op：`（ CF 转 ObjC 的时候使用）告诉编译器转移 CF 对象的所有权，开发者不再需要在CF一端负责释放对象

## 参考
[Concepts in Objective-C Programming](https://developer.apple.com/library/ios/documentation/General/Conceptual/CocoaEncyclopedia/Toll-FreeBridgin/Toll-FreeBridgin.html)
[Core Foundation Design Concepts](https://developer.apple.com/library/ios/documentation/corefoundation/Conceptual/CFDesignConcepts/Articles/tollFreeBridgedTypes.html#//apple_ref/doc/uid/TP40010677)
[Toll Free Bridging Internals](https://mikeash.com/pyblog/friday-qa-2010-01-22-toll-free-bridging-internals.html)
[Clang documentation: Objective-C Automatic Reference Counting (ARC)](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#bridged-casts)

