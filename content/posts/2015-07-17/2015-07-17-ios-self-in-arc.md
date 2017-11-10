---
author: null
categories:
- IOS
date: '2015-07-17'
description: null
photos: null
tags:
- arc
title: ios self in arc
---

[转自](http://blog.sunnyxx.com/2015/01/17/self-in-arc/)

## ARC对self的内存管理

记录下前两天的一次讨论，源于网络库`YTKNetwork`中“YTKRequest.m”的 `- start` 方法其中的几行代码：

```
- (void)start {
    // ......
    YTKRequest *strongSelf = self;
    [strongSelf.delegate requestFinished:strongSelf];
    if (strongSelf.successCompletionBlock) {
        strongSelf.successCompletionBlock(strongSelf);
    }
    [strongSelf clearCompletionBlock];
}
```

具体的问题大概是这样：

1. 调用方（如view controller）实例化并强引用YTKRequest对象，将自己作为其delegate
2. 调用方调用YTKRequest的 `- start` 方法发起网络请求
3. 调用方在 `- requestFinished:` 中执行了`self.request = nil`;
4. YTKRequest中，`- start`方法在回调完-` requestFinished:` 后 BAD_ACCESS了

也就是说，`- start`方法还未返回时，`self`就被外部释放了。作者发现了这个潜在的问题，所以在方法局部增设了一个`strongSelf`的强引用来保证self的生命周期延续到方法结束。问题是解决了，但是更希望知道原因。

简化说明就是：

```
- (void)foo {
    // self被delegate持有
    [self.delegate callout]; // 外部释放了这个对象
    // 这里self野指针
}
```

现在想想还是比较不符合常理，入参的self居然不能保证这个函数执行完成。后来查阅了下文档，发现是ARC的(gao)机(de)制(gui)，clang的《这篇ARC文档》中有明确的解释，总结如下：

- ARC下，self既不是strong也不是weak，而是unsafe_unretained的，也就是说，入参的self被表示为：（init系列方法的self除外）

```
- (void)start {
   const __unsafe_unretained YTKRequest *self;
   // ...
}
```

- 在方法调用时，ARC不会对self做retain或release，生命周期全由它的调用方来保证，如果调用方没有保证，就会出现上面的crash
- ARC这样做的原因是性能优化，objc中100%的方法（不是函数）调用第一个参数都是self，同时，99%的情况下，调用方都不会在方法执行时把这个对象释放，所以相比于在每个方法中插入对self的引用计数管理：
```
- (void)start {
    objc_retain(self);
    // 其中的代码self一定不会被释放
    objc_release(self);
}
```

优化了的性能还真是比较可观。 而且，ARC也用了挺多方法来避免开发者进行额外的引用计数控制，比如方法的命名约定，通过判断方法是否以如init，alloc，new，copy等关键字开头来决定其内存管理方式。

---

#### One more thing

在写test时发现，下面两种调用方法会导致不同结果：

```
- (void)viewDidLoad {
    // 1
    [_request start]; // crash
    // 2
    [self.request start]; // 正常
}
```

因为self.request是一次方法调用，返回的结果被objc_retainAutoreleasedReturnValue方法在局部进行了一次强引用