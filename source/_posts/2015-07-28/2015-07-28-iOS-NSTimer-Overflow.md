title: iOS NSTimer Overflow
categories: IOS
tags:
  - NSTimer
date: 2015-07-28 21:41:45
author:
description:
photos:
---
## NSTimer

### fire

我们先用 NSTimer 来做个简单的计时器，每隔5秒钟在控制台输出 Fire 。比较想当然的做法是这样的：
```
@interface DetailViewController ()
@property (nonatomic, weak) NSTimer *timer;
@end

@implementation DetailViewController
- (IBAction)fireButtonPressed:(id)sender {
    _timer = [NSTimer scheduledTimerWithTimeInterval:3.0f
                                              target:self
                                            selector:@selector(timerFire:)
                                            userInfo:nil
                                             repeats:YES];
    [_timer fire];
}

-(void)timerFire:(id)userinfo {
    NSLog(@"Fire");
}
@end
```

运行之后确实在控制台每隔3秒钟输出一次 `Fire` ，然而当我们从这个界面跳转到其他界面的时候却发现：控制台还在源源不断的输出着 Fire 。看来 Timer 并没有停止。

### invalidate
既然没有停止，那我们在 DemoViewController 的 dealloc 里加上 invalidate 的方法：
```
-(void)dealloc {
    [_timer invalidate];
    NSLog(@"%@ dealloc", NSStringFromClass([self class]));
}
```

再次运行，还是没有停止。原因是 Timer 添加到 Runloop 的时候，会被 Runloop 强引用：

> Note in particular that run loops maintain strong references to their timers, so you don’t have to maintain your own strong reference to a timer after you have added it to a run loop.

然后 Timer 又会有一个对 Target 的强引用（也就是 self ）：

> Target is the object to which to send the message specified by aSelector when the timer fires. The timer maintains a strong reference to target until it (the timer) is invalidated.

也就是说 `NSTimer` 强引用了 `self` ，导致 `self` 一直不能被释放掉，所以也就走不到 `self` 的 `dealloc` 里。

既然如此，那我们可以再加个 invalidate 按钮：
```
- (IBAction)invalidateButtonPressed:(id)sender {
    [_timer invalidate];
}
```

在 `invalidate` 方法的文档里还有这这样一段话：

> You must send this message from the thread on which the timer was installed. If you send this message from another thread, the input source associated with the timer may not be removed from its run loop, which could prevent the thread from exiting properly.
> NSTimer 在哪个线程创建就要在哪个线程停止，否则会导致资源不能被正确的释放。看起来各种坑还不少。

### dealloc
那么问题来了：如果我就是想让这个 NSTimer 一直输出，直到 DemoViewController 销毁了才停止，我该如何让它停止呢？

- NSTimer 被 Runloop 强引用了，如果要释放就要调用 invalidate 方法。
- 但是我想在 DemoViewController 的 dealloc 里调用 invalidate 方法，但是 self 被 NSTimer 强引用了。
- 所以我还是要释放 NSTimer 先，然而不调用 invalidate 方法就不能释放它。
- 然而你不进入到 dealloc 方法里我又不能调用 invalidate 方法。


### 解决方案
```
//
//  NSTimer+EZ_Helper.m
//  EZToolKit
//
//  Created by yangjun zhu on 15/5/20.
//  Copyright (c) 2015年 Cactus. All rights reserved.
//

#import "NSTimer+EZ_Helper.h"

@implementation NSTimer (EZ_Helper)
+ (NSTimer *)ez_scheduledTimerWithTimeInterval:(NSTimeInterval)inTimeInterval block:(void (^)())inBlock repeats:(BOOL)inRepeats
{
    void (^block)() = [inBlock copy];
    NSTimer * timer = [self scheduledTimerWithTimeInterval:inTimeInterval target:self selector:@selector(__executeTimerBlock:) userInfo:block repeats:inRepeats];
    return timer;
}

+ (NSTimer *)ez_timerWithTimeInterval:(NSTimeInterval)inTimeInterval block:(void (^)())inBlock repeats:(BOOL)inRepeats
{
    void (^block)() = [inBlock copy];
    NSTimer * timer = [self timerWithTimeInterval:inTimeInterval target:self selector:@selector(__executeTimerBlock:) userInfo:block repeats:inRepeats];
    return timer;
}

+ (void)__executeTimerBlock:(NSTimer *)inTimer;
{
    if([inTimer userInfo])
    {
        void (^block)() = (void (^)())[inTimer userInfo];
        block();
    }
}
@end
```



























