---
layout: post
title: "ios 时间处理"
date: 2015-05-27 15:49:48 +0800
comments: true
categories: IOS
---
### 时间

#### NSTimer
    NSTimer *time = [NSTimer scheduledTimerWithInterval:1.0
                        target:self
                        selector:@selector(update)
                        userInfo:nil
                        repeats:YES]
    [time invalidate] //关闭方法

<!--more-->

#### 时间间隔
    // The time interval
    NSTimeInterval theTimeInterval = ...;
    // Get the system calendar
    NSCalendar *sysCalendar = [NSCalendar currentCalendar];
    // Create the NSDates
    NSDate *date1 = [[NSDate alloc] init];
    NSDate *date2 = [[NSDate alloc] initWithTimeInterval:theTimeInterval sinceDate:date1];
    // Get conversion to months, days, hours, minutes
    unsigned int unitFlags = NSHourCalendarUnit | NSMinuteCalendarUnit | NSDayCalendarUnit | NSMonthCalendarUnit;
    NSDateComponents *breakdownInfo = [sysCalendar components:unitFlags fromDate:date1  toDate:date2  options:0];
    NSLog(@"Break down: %ld min : %ld hours : %ld days : %ld months", [breakdownInfo minute], 
        [breakdownInfo hour], [breakdownInfo day], [breakdownInfo month]);

#### 延迟执行
    @weakify(self);
    int64_t delayInSeconds = 1.0;
    dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, delayInSeconds * NSEC_PER_SEC);
    //performSelector方法
    [self performSelector:@selector(delayMethod) withObject:nil afterDelay:1.0f];

##### 1. 定时器:NSTimer
    [NSTimer scheduledTimerWithTimeInterval:1.0f target:self selector:@selector(delayMethod) userInfo:nil repeats:NO];

##### 2. leep方式
    [NSThread sleepForTimeInterval:1.0f]; [self delayMethod];

##### 3. GCD方式
    void RunBlockAfterDelay(NSTimeInterval delay, void (^block)(void)) {
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(NSEC_PER_SEC*delay),
        dispatch_get_current_queue(), block);
    }

##### 4. 使用NSOperationQueue 在应用程序的下一个主循环执行
    [[NSOperationQueue mainQueue] addOperationWithBlock:aBlock];