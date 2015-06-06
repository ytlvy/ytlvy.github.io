---
layout: post
title: "全局样式"
date: 2015-05-27 15:48:33 +0800
comments: true
categories: IOS
---
### 全局样式
    - (void)setStyle{
        [[UIApplication sharedApplication] setStatusBarStyle: UIStatusBarStyleLightContent];

        [[UIBarButtonItem appearanceWhenContainedIn:[UINavigationBar class], nil] setTitleTextAttributes:
         @{NSForegroundColorAttributeName:[UIColor blackColor],
           NSFontAttributeName: [UIFont preferredFontForTextStyle:UIFontTextStyleHeadline]
           } forState:UIControlStateNormal];

        [[UIBarButtonItem appearanceWhenContainedIn:[UINavigationBar class], nil] setTitleTextAttributes:
         @{NSForegroundColorAttributeName:[UIColor whiteColor],
           NSFontAttributeName: [UIFont preferredFontForTextStyle:UIFontTextStyleHeadline]
           } forState:UIControlStateHighlighted];

        // [[UINavigationBar appearance] setBarTintColor:UIColorFromRGB(0xFC6259)];

        // Uncomment to change the color of back button
        [[UINavigationBar appearance] setTintColor:[UIColor whiteColor]];

        // Uncomment to assign a custom backgroung image
        // [[UINavigationBar appearance] setBackgroundImage:[UIImage imageNamed:@"NavBar-Background"] forBarMetrics:UIBarMetricsDefault];

        // Uncomment to change the back indicator image
        // [[UINavigationBar appearance] setBackgroundColor:[UIColor blackColor]];
        // [[UINavigationBar appearance] setBackIndicatorTransitionMaskImage:[UIImage imageNamed:@""]];

        // Uncomment to change the font style of the title
        NSShadow *shadow = [[NSShadow alloc] init];
        shadow.shadowColor = [UIColor colorWithRed:0.0 green:0.0 blue:0.0 alpha:0.8];
        shadow.shadowOffset = CGSizeMake(0, 1);

        [[UINavigationBar appearance] setTitleTextAttributes: [NSDictionary dictionaryWithObjectsAndKeys:[UIColor colorWithRed:245.0/255.0 green:245.0/255.0 blue:245.0/255.0 alpha:1.0], NSForegroundColorAttributeName,shadow, NSShadowAttributeName,[UIFont preferredFontForTextStyle:UIFontTextStyleHeadline], NSFontAttributeName, nil]];
        [[UINavigationBar appearance] setTitleVerticalPositionAdjustment:0.0 forBarMetrics:UIBarMetricsDefault];
    }