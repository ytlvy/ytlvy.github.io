---
layout: post
title: "<Pro.Multithreading.and.Memory.Management.for.iOS.and.OS.X>笔记"
date: 2015-05-27 15:55:30 +0800
comments: true
categories: IOS
---
# 读书笔记

## weake __unsafe_unretained
    id __weake obj = [[NSObject alloc] init];
    //obj为nil，因为weak不能持有对象，所以生成后马上释放
    id __unsafe_unretained obj = [[NSObject alloc] init];
    //obj is nil

    NSAutoReleasePool 对象不能调用 autorelease 方法 因为NSAutoReleasePool类重载了该方法

## AutoReleasePool
    //arc
    @autoreleasepool{
        id __autoreleasing obj2;
    }

