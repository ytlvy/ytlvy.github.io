---
layout: post
title: "IOS8 定位服务权限的问题"
date: 2015-05-27 15:49:57 +0800
comments: true
categories: IOS
---
## IOS8 定位服务权限的问题
    self.manger = [[CLLocationManager alloc]init];
    self.manger.distanceFilter = kCLDistanceFilterNone; // meters
    self.manger.delegate = self;
    [self.manger requestAlwaysAuthorization];
    [self.manager requestWhenInUseAuthorization];
    self.manger.desiredAccuracy = kCLLocationAccuracyBestForNavigation;
    [self.manger startUpdatingLocation];

    //并且在info.plist文件中增加
    NSLocationWhenInUseUsageDescription  BOOL YES
    NSLocationAlwaysUsageDescription         string “提示描述”
    两个字段，在iOS8中才能进行正确的获取服务权限