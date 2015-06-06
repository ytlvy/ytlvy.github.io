---
layout: post
title: "ios 多线程"
date: 2015-05-27 15:46:35 +0800
comments: true
categories: IOS
---
#### dispatch_queue_create
    dispatch_queue_create生成的queue需要手动内存管理
    dispatch_release
    dispatch_retain
    dispatch_queue_t myConcurrentDispatchQueue = dispatch_queue_create( "com.example.gcd.MyConcurrentDispatchQueue", DISPATCH_QUEUE_CONCURRENT);
    dispatch_queue_t mySerialDispatchQueue = dispatch_queue_create("com.example.gcd.MySerialDispatchQueue", NULL);
    global dispatch queues. They are concurrent dispatch queues that are usable anywhere in the application

<!--more-->

    By the way, if you call the dispatch_retain or dispatch_release function on the main dispatch queue or on a global dispatch queue, nothing happens.

    Please note that the dispatch_after function doesn’t execute the task after the specified time. After the time, it adds the task to the dispatch queue

    dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull * NSEC_PER_SEC);
    dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 150ull * NSEC_PER_MSEC);//millisecond unit values

    dispatch_time_t getDispatchTimeByDate(NSDate *date) {
        NSTimeInterval interval; 
        double second, subsecond; 
        struct timespec time; 
        dispatch_time_t milestone;
        interval = [date timeIntervalSince1970]; 
        subsecond = modf(interval, &second); 
        time.tv_sec = second;
        time.tv_nsec = subsecond * NSEC_PER_SEC; 
        milestone = dispatch_walltime(&time, 0);
        return milestone; 
    }

#### 多个任务完成后执行
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_group_t group = dispatch_group_create();
    dispatch_group_async(group, queue, ^{NSLog(@"blk0");}); dispatch_group_async(group, queue, ^{NSLog(@"blk1");}); dispatch_group_async(group, queue, ^{NSLog(@"blk2");});
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{NSLog(@"done");}
    );
    dispatch_release(group);