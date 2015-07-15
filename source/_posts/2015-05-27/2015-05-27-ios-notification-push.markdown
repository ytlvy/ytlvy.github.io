---
layout: post
title: "ios Notification push"
date: 2015-05-27 15:49:05 +0800
comments: true
categories: IOS
---
### Event
#### 注册一个消息中心
    NSString *const GameToIPhoneNotification = @"GameToIPhoneNotification"; 
    NSNotificationCenter *center = [NSNotificationCenter defaultCenter];
    [center addObserver:self selector:@selector(onToIphone:) name:GameToIPhoneNotification object:nil];
    -(void)onToIphone:(NSNotification*)notify

#### 调用信息
    NSNotificationCenter * center = [NSNotificationCenter defaultCenter];   
    [center postNotificationName:GameToIPhoneNotification object:nil userInfo:[NSDictionary dictionaryWithObjectsAndKeys: [NSNumber numberWithInt:SMSRecommendNotification] , @"actcode",nil]];
    
    [NSDictionary dictionaryWithObjectsAndKeys: [NSNumber numberWithInt:SMSRecommendNotification] 这个是传递给-(void)onToIphone:(NSNotification*)notify 的参数    

<!--more-->

### 推送
    //ios 7
    - (void)application:(UIApplication *)app didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken {
        NSString *token = [NSString stringWithFormat:@"%@", deviceToken];
        DDLogInfo(@"push deviceToken: %@", token);
    }
    - (void)application:(UIApplication *)app didFailToRegisterForRemoteNotificationsWithError:(NSError *)error {
        NSLog(@"get push deviceToken error%@", error);
    }
    //ios8
    - (void)application:(UIApplication *)application
    didRegisterUserNotificationSettings:(UIUserNotificationSettings *)notificationSettings{
        [application registerForRemoteNotifications];
    }

    //For interactive notification only
    - (void)application:(UIApplication *)application handleActionWithIdentifier:(NSString *)identifier 
                forRemoteNotification:(NSDictionary *)userInfo completionHandler:(void(^)())completionHandler {
        //handle the actions
        if ([identifier isEqualToString:@"declineAction"]){
        }
        else if ([identifier isEqualToString:@"answerAction"]){
        }
    }