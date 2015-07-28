title: iOS ARC Practices
categories: IOS
tags:
  - ARC
date: 2015-07-28 21:39:38
author:
description:
photos:
---
[转自](http://amattn.com/p/arc_best_practices.html)

## ARC 最佳实践

### General

1) 数值变量应该使用 assign：
```
@property (nonatomic, assign) int scalarInt;
@property (nonatomic, assign) CGFloat scalarFloat;
@property (nonatomic, assign) CGPoint scalarStruct;
```

2) 在层次结构上属于下一级的对象应该使用 strong ：
```
@property (nonatomic, strong) id childObject;
```

3) 在层次结构上属于上一级的对象应该使用 weak ，另外，当出现循环引用的时候也应该使用 weak ：
```
@property (nonatomic, weak) id parentObject;
@property (nonatomic, weak) NSObject <SomeDelegate> *delegate;
```

4) 闭包应该使用 copy ：

```
@property (nonatomic, copy) SomeBlockType someBlock;
```

5) 在 dealloc 里：

- 从观察者中移除 (remove observers)
- 取消订阅通知 (unregister for notifications)
- 设置非 weak 的委托为 nil (set any non-weak delegates to nil)
- 关闭所有的计时器 (invalidate any timers)

6) 所有的 IBOutlet 都应该是 weak 的。除非顶层的 IBOutlet 应该是 strong 的，比如 UIViewController 的 View 是需要直接拥有的，所以应该设置成 strong 

<!-- more -->
### Bridging

从文档来看是这样的：
```
id my_id;
CFStringRef my_cfref;
NSString   *a = (__bridge NSString*)my_cfref;     // Noop cast.
CFStringRef b = (__bridge CFStringRef)my_id;      // Noop cast.
NSString   *c = (__bridge_transfer NSString*)my_cfref; // -1 on the CFRef
CFStringRef d = (__bridge_retained CFStringRef)my_id;  // returned CFRef +1
```

简单的翻译成大白话：
- __bridge 只做类型转换，但是不修改对象内存的管理权。
- __bridge_transfer 将 Core Foundation 的对象转换为 Objective-C 的对象，同时将对象内存的管理权交给 ARC 。 ARC 会自动将引用计数减1 。
- __bridge_retained 将 Objective-C 的对象转换为 Core Foundation 的对象，同时将对象内存的管理权交给我们，引用计数会自动加1。后续需要使用 CFRelease 或者相关方法来释放对象。

### NSError
无所不在的 `NSError` 有点特殊，在标准的 Cocoa 使用中， `NSError` 是通过外部参数 `(out-parameter)` 实现的，又称为 `indirect pointer` 。

在 ARC 里， `out-parameter` 默认是 `__autoreleasing` 的，并且实现方法应该是这样的：
```
- (BOOL)performWithError:(__autoreleasing NSError **)error {
    // 如果发生了一些错误
    if (error) {
        // 写入到外部参数， ARC 会自动释放
        *error = [[NSError alloc] initWithDomain:@"" 
                                            code:-1 
                                        userInfo:nil];
        return NO;
    }
    else {
        return YES;
    }
}
```

当我们使用这个方法的时候， *error 前面应该加上 __autoreleasing 的标记：
```
NSError __autoreleasing *error = error;
BOOL OK = [myObject performOperationWithError:&error];
if (!OK)
{
    // handle the error.
}
```

如果你标记了 `__autoreleasing` ，编译器会简单的插入一个临时的中介对象。这是为了能够向下兼容的妥协方案，因为有些编译器的设置并没有自动把它们设置为 `__autoreleasing` ，所以为了安全起见，还是加上 `__autoreleasing` 的比较保险。

### @autoreleasepool
可以在如下场景使用自动释放池：

- 遍历很多很多遍的时候
- 创建大量的临时变量

比如下面这个例子：
```
// 如果 someArray 里的元素非常多
for (id obj in someArray) {
    @autoreleasepool {
        // 或者在每次遍历里都创建了大量的临时变量
    }
}
```

我们可以通过 `@autoreleasepool` 标记创建和销毁一个自动释放池。


### Blocks
总的来说，闭包用起来还行，但是还是有那么点小问题。

当我们把闭包添加到集合中的时候 (比如 NSArray) ，一定要先 copy 一下：

```
someBlockType someBlock = ^{NSLog(@"hi");};
[someArray addObject:[someBlock copy]];
```

循环引用是十分危险的，你可能看到过这些 warning ：
```
warning: capturing 'self' strongly in this 
block is likely to lead to a retain cycle 
[-Warc-retain-cycles,4]

SomeBlockType someBlock = ^{
    [self someMethod];
};
```

警告的原因是 `someBlock` 是 `self` 的强引用，而 `someBlock` 又捕捉并且 `retain` 了 `self。`

再举一个表现不明显的循环引用：任何变量的父对象都是会被捕捉的：
```
// 下面这个闭包会持有 "self"
SomeBlockType someBlock = ^{
    BOOL isDone = _isDone;  // _isDone 是 self 的一个变量
};
```

更安全但是也更啰嗦的方法是使用 weakSelf ：
```
__weak SomeObjectClass *weakSelf = self;

SomeBlockType someBlock = ^{
    SomeObjectClass *strongSelf = weakSelf;
    if (strongSelf == nil)
    {
        // The original self doesn't exist anymore.
        // Ignore, notify or otherwise handle this case.
    }
    else
    {
        [strongSelf someMethod];
    }
};
```

有时候你也需要注意避免对象的循环引用：如果 someObject 将会被 block 强引用，可以用 weakSomeObject 打破循环引用：
```
SomeObjectClass *someObject = ...
__weak SomeObjectClass *weakSomeObject = someObject;

someObject.completionHandler = ^{
    SomeObjectClass *strongSomeObject = weakSomeObject;
    if (strongSomeObject == nil)
    {
        // The original someObject doesn't exist anymore.
        // Ignore, notify or otherwise handle this case.
    }
    else
    {
        // okay, NOW we can do something with someObject
        [strongSomeObject someMethod];
    }
};
```


### Accessing CGThings from NSThings or UIThings
先看一段代码：
```
UIColor *redColor = [UIColor redColor]; 
CGColorRef redRef = redColor.CGColor;
// 用 redRef 做一些事情
```

上面这段代码有一些很隐蔽的问题。当你创建 `redRef` 的时候，如果 `redColor` 不再被使用，那么 `redColor` 会在备注那一行立即被销毁。

问题是 `redColor` 持有了 `redRef` ，当访问 `redRef` 的时候，它有可能是也有可能不再是一个 `colorRef` 。更糟糕的是，这种问题很少在虚拟机上出现，这更有可能在那种低内存的设备上出现，比如第一代的 `iPad` 老爷机。

解决的方案也有很多。最关键的一点是：在使用 `redRef` 的时候，我们需要保证 `redColor` 是有效的。

一种最简单的方式就是使用 `__autoreleasingpool` ：
```
UIColor * __autoreleasing redColor = [UIColor redColor];
CGColorRef redRef = redColor.CGColor;
```

注意这时候 `redColor` 在方法返回之前都不会被释放掉，所以我们可以放心的在当前的方法里使用 `redRef` 。

另一种方法是 `retain` 一下 `redRef` ：
```
UIColor *redColor = [UIColor redColor];
CGColorRef redRef = CFRetain(redColor.CGColor);
// 尽情的使用 redRef ，注意，在结束的时候要 release
CFRelease(redRef)
```

注意：我们需要在 `redColor.CGColor` 处使用 `CFRetain()`，因为在最近一次使用之后 `redColor` 就被释放了。比如下面这段代码就是没啥用的：
```
UIColor *redColor = [UIColor redColor];
CGColorRef redRef = redColor.CGColor; // redColor 在这行代码之后就被 release 了
CFRetain(redRef);  // 这里可能会崩溃
```

有趣的是标记了 这里可能会崩溃 的那一行。在我的印象里，在虚拟机里很少崩溃，而在真机上就很常见。

### 单例模式
标准的单例模式的实现应该是这个样子的：
```
+ (MyClass *)singleton
{
    static MyClass *sharedMyClass = nil;
    static dispatch_once_t once = 0;
    dispatch_once(&once, ^{sharedMyClass = [[self alloc] init];});
    return sharedMyClass;
}
```

有时候我们还需要销毁单例，不过如果不是用在单元测试里，那么很有可能你不该使用单例模式。

可以销毁并重建的单例模式是这样的：
```
// 在 singleton 外声明一个静态变量
static MyClass *__sharedMyClass = nil;

+ (MyClass *)singleton
{
    static dispatch_once_t once = 0;
    dispatch_once(&once, ^{__sharedMyClass = [[self alloc] init];});
    return __sharedMyClass;
}

// 仅可用于测试！
- (void)destroyAndRecreateSingleton
{
    __sharedMyClass = [[self alloc] init];
}
```










