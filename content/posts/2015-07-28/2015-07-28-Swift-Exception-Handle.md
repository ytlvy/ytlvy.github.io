---
author: null
categories:
- Swift
date: '2015-07-28'
description: null
photos: null
tags:
- Swift
- Exception
title: Swift Exception Handle
---

[转自](http://blog.callmewhy.com/2015/04/20/error-handling-in-swift/)

## Swift 中的异常处理

### 问题
在开发过程中，异常处理算是比较常见的问题了。

举一个比较常见的例子：用户修改注册的邮箱，大概分为以下几个步骤：

- 接收到一个用户的请求：我要修改邮箱地址
- 验证一下请求是否合法，将请求进行格式转化
- 更新以前的邮箱地址记录
- 给新的邮箱地址发送验证邮件
- 将结果返回给用户

上面的步骤如果一切顺利，那代码肯定干净利落，但是人生不如意十有八九，上面的步骤很容易出现问题：

- 用户把邮箱地址填成了家庭地址
- 用户是个黑客，没登录就发送了更新请求
- 发送验证邮件的时候服务器爆炸了，发送邮件失败

各种异常都会导致这次操作的失败。


### 方案一
在传统的处理方案里，一般是遇到异常就往上抛：
![](http://segmentfault.com/img/bVlsK9)

这种方案想必大家都不陌生，比如下面这段代码：
```
NSError *err = nil;
CGFloat result = [MathTool divide:2.5 by:3.0 error:&err];

if (err) {
    NSLog(@"%@", err)
} else {
    [MathTool doSomethingWithResult:result]
}
```

<!--more-->

### 方案二
而另一种方案，则是将错误的结果继续往后传，在最后统一处理：
![](http://segmentfault.com/img/bVlsLe)

这种方案有两个问题：
- 在发生异常的时候，如何把异常继续传给下面的函数？
- 当整个流程结束的时候，一个函数如何输出多个结果？

### 车轨
我们把方案二抽象出来，就像是一段车轨：

![](http://segmentfault.com/img/bVlsNx)

对于同一个输入，会有 Success 和 Failure 两种输出结果，对于 Success 的情况，我们希望它能继续走到后面的流程里，而对于 Failure 的情况，它怎么处理并不重要，我们希望它能避开后面的流程：

![](http://segmentfault.com/img/bVlsNC)

于是乎，两段车轨拼接的时候，便成了这样：

![](http://segmentfault.com/img/bVlsNH)

那么三段什么的自然也不在话下了。我们把下面那根 Failure 的线路扩展一下，便会看到两条平行的线路，这便是“双轨模型” (Two Track Model) ，这是用“面向轨道编程”思想解决异常处理的理论基础。

![](http://segmentfault.com/img/bVlsNK)

这就是 “面向轨道编程” 。一开始我觉得这概念应该只是来搞笑的，仔细想想似乎倒也是很贴切。将事件流当做两条平行的轨道，如果顺利则在上行轨道，继续传递给下个业务逻辑去处理，如果出现异常也不慌，直接扔到下行轨道，一直在下行轨道传递到终点，在最后统一处理。

这样处理使得整个流程变成了一条双进双出的流水线，有点像是 shell 里的 pipeline ，上一次的输出作为下一次的输入，十分顺畅。而且拼接起来也很方便，我们可以把三段拼接成一段暴露给其他对象使用：

![](http://segmentfault.com/img/bVlsOy)


### 实现

接下来看看在 Swift 中如何应用这种思路处理异常。

首先我们需要两种类型的输出结果：

- 成功，返回某种类型的值
- 失败，返回一个 Error 对象或者失败的具体信息


照着这个想法，我们可以定义一个 Result 枚举用做输出：
```
enum Result<T> {
    case Success(T)
    case Failure(String)
}
```


利用 Swift 的枚举特性，我们可以在成功的枚举值里关联一些返回值，然后在失败的情况下则带上失败的消息内容。不过 enum 目前还不支持泛型，我们可以在外面封装一个 Box 类来解决这个问题：
```
final class Box<T> {
    let value: T
    init(value: T) {
        self.value = value
    }
}

enum Result<T> {
    case Success(Box<T>)
    case Failure(String)
}
```

再看下一开始我们举的那个例子，用这个枚举类重新写下就是这样的：
```
var result = divide(2.5, by:3)
switch result {
case .Success(let value):
    doSomethingWithResult(value)
case .Failure(let errString):
    println(errString)
}

```

“看起来好像也没什么嘛，你不还是用了个大括号处理两种情况嘛！”（嫌弃脸

确实正如这位热情的朋友所说，写完这个例子我也没觉得有什么优点，难道我就是来搞笑的？

“并不。”（严肃脸


### 栗子
接下来我们举个栗子玩一玩。为了更好的观赏效果，请允许我使用浮夸的写法和粗暴的命名举这个栗子。

比如对于即将输入的数字 x ，我们希望输出 `4 / (2 / x - 1)` 的计算结果。这里会有两处出错的可能，一个是 `(2 / x)` 时 x 为 0 ，另一个就是 `(2 / x - 1)` 为 0 的情况。

先看下传统写法：
```
let errorStr = "输入错误，我很抱歉"
func cal(value: Float) {
    if value == 0 {
        println(errorStr)
    } else {
        let value1 = 2 / value
        let value2 = value1 - 1
        if value2 == 0 {
            println(errorStr)
        } else {
            let value3 = 4 / value2
            println(value3)
        }
    }
}
cal(2)    // 输入错误，我很抱歉
cal(1)    // 4.0
cal(0)    // 输入错误，我很抱歉
```

那么用面向轨道的思想怎么去解决这个问题呢？

大概是这个样子的：

```
final class Box<T> {
    let value: T
    init(value: T) {
        self.value = value
    }
}

enum Result<T> {
    case Success(Box<T>)
    case Failure(String)
}

let errorStr = "输入错误，我很抱歉"

func cal(value: Float) {
    func cal1(value: Float) -> Result<Float> {
        if value == 0 {
            return .Failure(errorStr)
        } else {
            return .Success(Box(value: 2 / value))
        }
    }
    func cal2(value: Result<Float>) -> Result<Float> {
        switch value {
        case .Success(let v):
            return .Success(Box(value: v.value - 1))
        case .Failure(let str):
            return .Failure(str)
        }
    }
    func cal3(value: Result<Float>) -> Result<Float> {
        switch value {
        case .Success(let v):
            if v.value == 0 {
                return .Failure(errorStr)
            } else {
                return .Success(Box(value: 4 / v.value))
            }
        case .Failure(let str):
            return .Failure(str)
        }
    }

    let r = cal3(cal2(cal1(value)))
    switch r {
    case .Success(let v):
        println(v.value)
    case .Failure(let s):
        println(s)
    }   
}
cal(2)    // 输入错误，我很抱歉
cal(1)    // 4.0
cal(0)    // 输入错误，我很抱歉
```

同学，放下手里的键盘，冷静下来，有话好好说。

### 反思
面向轨道之后，代码量翻了两倍多，而且似乎变得更难读了。浪费了大家这么多时间结果就带来这么个玩意儿，实在是对不起观众们热情的掌声。

仔细看下上面的代码， switch 的操作重复而多余，都在重复着把 Success 和 Failure 分开的逻辑，实际上每个函数只需要处理 Success 的情况。我们在 Result 中加入 funnel 提前处理掉 Failure 的情况：

```
enum Result<T> {
    case Success(Box<T>)
    case Failure(String)

    func funnel<U>(f:T -> Result<U>) -> Result<U> {
        switch self {
        case Success(let value):
            return f(value.value)
        case Failure(let errString):
            return Result<U>.Failure(errString)
        }
    }
}
```

`funnel` 帮我们把上次的结果进行分流，只将 `Success` 的轨道对接到了下个业务上，而将 `Failure` 引到了下一个 `Failure` 轨道上。

接下来再回到栗子里，此时我们已经不再需要传入 `Result` 值了，只需要传入 `value` 即可：

```
func cal(value: Float) {
    func cal1(v: Float) -> Result<Float> {
        if v == 0 {
            return .Failure(errorStr)
        } else {
            return .Success(Box(2 / v))
        }
    }

    func cal2(v: Float) -> Result<Float> {
        return .Success(Box(v - 1))
    }

    func cal3(v: Float) -> Result<Float> {
        if v == 0 {
            return .Failure(errorStr)
        } else {
            return .Success(Box(4 / v))
        }
    }

    let r = cal1(value).funnel(cal2).funnel(cal3)
    switch r {
    case .Success(let v):
        println(v.value)
    case .Failure(let s):
        println(s)
    }
}
```


看起来简洁了一些。我们可以通过 `cal1(value).funnel(cal2).funnel(cal3)` 这样的链式调用来获取计算结果。

“面向轨道”编程确实给我们提供了一个很有趣的思路。本文只是一个简单地讨论，进一步学习可以仔细阅读后面的参考文献。比如 ValueTransformation.swift 这个真实的完整案例，以及 antitypical/Result 这个封装完整的 Result 库。文中的实现方案只是一个比较简单的方法，和前两种实现略有差异。

面向铁轨，春暖花开。愿每段代码都走在 Happy Path 上，愿每个人都有个 Happy Ending 。
md