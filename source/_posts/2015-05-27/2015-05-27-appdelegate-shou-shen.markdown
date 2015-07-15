---
layout: post
title: "Appdelegate瘦身"
date: 2015-05-27 15:47:55 +0800
comments: true
categories: IOS
---
- (BOOL)application:(UIApplication *)application
didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // ...
    [FooModule setup];
    [[BarModule sharedInstance] setup];
    // ...
    return YES;
}

/// FooModule.m
+ (void)load
{
    __block id observer =
    [[NSNotificationCenter defaultCenter]
     addObserverForName:UIApplicationDidFinishLaunchingNotification
     object:nil
     queue:nil
     usingBlock:^(NSNotification *note) {
         [self setup]; // Do whatever you want
         [[NSNotificationCenter defaultCenter] removeObserver:observer];
     }];
}

// 解释：
// + load方法在足够早的时间点被调用
// block版本的通知注册会产生一个__NSObserver *对象用来给外部remove观察者
// block对observer对象的捕获早于函数的返回，所以若不加__block，会捕获到nil
// 在block执行结束时移除observer，无需其他清理工作
// 这样，在模块内部就完成了在程序启动点代码的挂载