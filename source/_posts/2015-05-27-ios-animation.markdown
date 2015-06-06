---
layout: post
title: "ios 动画"
date: 2015-05-27 15:48:24 +0800
comments: true
categories: IOS
---
#### 动画保持结果
    fillMode = kCAFillModeForwards

#### CAKeyframeAnimation 动画使用 开始、暂停动画
##### 开始一个帧动画
```
- (void)showAlertAnimation {  
    CAKeyframeAnimation *animation = [CAKeyframeAnimation
                       animationWithKeyPath:@"transform"]; 

    //方式1：放大再缩小（类似系统alert）  
    //如果需要更好的效果  
    //可以添加 .keyTimes 属性。  
    NSMutableArray *values = [NSMutableArray array]; 
    [values addObject:
                [NSValue valueWithCATransform3D:CATransform3DMakeScale(0.1, 0.1, 1.0)]];
    [values addObject:
                [NSValue valueWithCATransform3D:CATransform3DMakeScale(1.2, 1.2, 1.0)]]; 
     [values addObject:
                [NSValue valueWithCATransform3D:CATransform3DMakeScale(0.9, 0.9, 0.9)]];
      [values addObject:
                [NSValue valueWithCATransform3D:CATransform3DMakeScale(1.0, 1.0, 1.0)]]; 
      animation.values = values; 

 
    //方式2:直接缩小  
    animation.fillMode = kCAFillModeForwards;  
    animation.removedOnCompletion = NO;  
    animation.duration = .3;  
      
    [self.alertView.layer addAnimation:animation forKey:@"showAlert"];  
}  
```

<!--more-->

> 其中：fillMode主要是决定显示layer在动画完成后的状态..一般和removedOnCompletion一起使用..
如果fillmode是..kCAFillModeRemoved 或..kCAFillModeBackwards...
不管removedOnCompletion是yes还是no,都会回到原始状态..
一般用在重复的动画里..比如图片旋转5圈..你做一圈的功能.然后重复5次..就行了..

> kCAFillModeForwards 或 kCAFillModeBoth模式下...
如果..removedOnCompletion 是yes,动画完成后会回到原始状态..
removedOnCompletion是NO的话..动画完成后会保持状态..
保持状态只是保持可见层(presentation)的状态...layer本身的状态不会改变.
比如...layer的frame,在动画完成后不管是否保持状态,frame都不变..完成后得手动设定frame的位置

##### 开始、暂停动画
```
-(void)pauseLayer:(CALayer*)layer   {  
    CFTimeInterval pausedTime = [layer convertTime:CACurrentMediaTime() fromLayer:nil];  
    layer.speed = 0.0;  
    layer.timeOffset = pausedTime;  
}  
//恢复layer上的动画  
-(void)resumeLayer:(CALayer*)layer   {  
    CFTimeInterval pausedTime = [layer timeOffset];  
    layer.speed = 1.0;  
    layer.timeOffset = 0.0;  
    layer.beginTime = 0.0;  
    CFTimeInterval timeSincePause = [layer convertTime:CACurrentMediaTime() fromLayer:nil] - pausedTime;  
    layer.beginTime = timeSincePause;  
}  
```

#### 普通动画效果
```
[UIView beginAnimations:@"mvoeToRight" context:nil];
sender.center = CGPointMake(250., 60.);
[UIView commitAnimations];
sender.transform = CGAffineTransformMakeRotation(M_PI);
```

#### 复杂动画
```
+(void)showAnimationType: (NSString *)type withSubType: (NSString *)subtype 
                duration: (CFTimeInterval)duration 
                timingFunction: (NSString *)timingFunction 
                        View: (UIView *)theView{
    CATransition *animation = [CATransition animation];
    //animation.delegate = self;
    animation.duration = duration;
    animation.timingFunction = [CAMediaTimingFunction functionWithName:timingFunction];
    animation.fillMode = KCAFillModeForwards;

    //各种动画效果
    //@"cube" @"moveIn" @"reveal" @"fade"(default) @"pageCurl" @"pageUnCurl" @"suckEffect" @"rippleEffect" @"oglFlip" @"rotate"

    animation.type = type;

    //动画方向
    //KCATransitionFade; KCATransitionMoveIn;KCATransitionPush;KCATransitionReveal;
    KCATransitionFromRight;KCATransitionFromLeft;KCATranstionFromTop;
    KCATransitionFromBottom;
    animation.subType = subType;
    [theView.layer addAnimation:animation forKey:@"nil"];
}
```

