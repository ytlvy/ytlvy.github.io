---
author: null
categories:
- IOS
date: '2015-08-08'
description: null
photos: null
tags:
- GCD
title: iOS Dispatch_async usages tips
---

[reference](http://blog.cnbluebox.com/blog/2014/10/20/dispatch-asyncyi-bu-de-xiao-miao-yong/)

## Dispatch_async异步的小妙用

GCD 对于cocoa和cocoa Touch的开发者来说已经是在熟悉不过了，相信有些基础的人都能说出一些特性。但是今天我们不是讲GCD的介绍和使用bulabulabula. 本文就介绍下使用`dispatch_async`的一个小妙用，且看下文分解。

我们都知道dispatch_async可以用来异步执行代码。你的第一反应的可能是后台、多线程 这样的字眼，但是异步执行不是就代表多线程，它只是代表代码的执行方式，知道这点可以做更多事情。

### afterDelay

你是否做过这样的事情，在修改一个bug时，把方法延迟执行，like this:

```
//[self doSomething];
[self performSelector:@selector(doSomething) withObject:nil afterDelay:0.1f];
```

Let me guess, 你一定是发现doSomething方法中所依赖的数据或环境还达不到，立即执行的话，可能会有问题。所以你就让doSomething延迟一点时间执行，保证它需要的数据或环境已经处理完成，改完打包测试，bug看起来已经修复了。

这个问题从大多数场景来说的确已经修复了，但是在强迫症眼里，好像还是有些瑕疵，不是完美方案。

两个问题，你怎么能100%确保你需要的数据或环境能再0.1s内准备完成呢； 另一个是数据准备完成到doSomething执行中间可能会有一段空白时间，有时是我们不希望的。那如何替代这个方案呢，如果你得doSomething需要的数据和环境是在主线程执行的，那么方案很简单：
```
//[self doSomething];
dispatch_async(dispatch_get_main_queue(), ^{
    [self doSomething];
});
```
你可能问了，我本来就在主线程执行任务，为什么还要往main_queue里面提交任务呢。巧妙的地方就在这，我们巧妙的把doSomething作为任务提交到主线程，它会等待主线程的任务一个个执行完成才执行，这样就可以巧妙的解决这个问题。

不过注意，不是所有场景都适合这个用法，就像上面描述的，你要确保doSomething的依赖条件实在主线程中执行的。



