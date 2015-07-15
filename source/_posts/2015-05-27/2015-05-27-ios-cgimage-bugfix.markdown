---
layout: post
title: "CGImage绘制倒置修复"
date: 2015-05-27 15:52:43 +0800
comments: true
categories: IOS
---
```
UIImage* mars = [UIImage imageNamed:@"Mars.png"];

CGSize sz = [mars size];

CGImageRef marsCG = [mars CGImage];

CGSize szCG = CGSizeMake(CGImageGetWidth(marsCG), CGImageGetHeight(marsCG));

CGImageRef marsLeft = CGImageCreateWithImageInRect(marsCG, CGRectMake(0,0,szCG.width/2.0,szCG.height));

CGImageRef marsRight = CGImageCreateWithImageInRect(marsCG, CGRectMake(szCG.width/2.0,0,szCG.width/2.0,szCG.height));

UIGraphicsBeginImageContextWithOptions(CGSizeMake(sz.width*1.5, sz.height), NO, 0);

[[UIImage imageWithCGImage:marsLeft scale:[mars scale] orientation:UIImageOrientationUp] drawAtPoint:CGPointMake(0,0)];

[[UIImage imageWithCGImage:marsRight scale:[mars scale] orientation:UIImageOrientationUp] drawAtPoint:CGPointMake(sz.width,0)];

UIImage* im = UIGraphicsGetImageFromCurrentImageContext();

UIGraphicsEndImageContext();

CGImageRelease(marsLeft); CGImageRelease(marsRight); 
```