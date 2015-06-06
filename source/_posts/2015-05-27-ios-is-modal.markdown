---
layout: post
title: "ios 是否模态弹出"
date: 2015-05-27 15:49:20 +0800
comments: true
categories: IOS
---
## 是否模态弹出
    - (BOOL)isModal {
         if([self presentingViewController])
             return YES;
         if([[self presentingViewController] presentedViewController] == self)
             return YES;
         if([[[self navigationController] presentingViewController] presentedViewController] == [self navigationController])
             return YES;
         if([[[self tabBarController] presentingViewController] isKindOfClass:[UITabBarController class]])
             return YES;
             
        return NO;
     }