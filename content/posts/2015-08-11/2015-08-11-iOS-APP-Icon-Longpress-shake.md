---
categories:
- IOS
date: '2015-08-11'
tags:
- Animation
title: iOS APP Icon Longpress shake
---

##  iOS APP图标长按抖动效果的实现（CAAnimation）
实现效果:当长按图标时，图标抖动，点击图标以外区域时停止抖动

![](http://img.blog.csdn.net/20150808181226925)

工程下载：[github工程下载](https://github.com/Nongchaozhe/iconShakeDemo)

1) 视图代码

```
@interface ViewController ()
{
    UIImageView *imageView;
}
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    imageView = [[UIImageView alloc] initWithFrame:CGRectMake(0, 0, 100, 100)];
    imageView.center = self.view.center;
    imageView.image = [UIImage imageNamed:@"icon1"];
    imageView.userInteractionEnabled = YES;
    [self.view addSubview:imageView];

    //添加长按手势
    UILongPressGestureRecognizer *recognize = [[UILongPressGestureRecognizer alloc] initWithTarget:self action:@selector(longPress)];
    //长按响应时间
    recognize.minimumPressDuration = 1;
    [imageView addGestureRecognizer:recognize];
    // Do any additional setup after loading the view, typically from a nib.
}
```

<!--more-->

2) 手势响应

```
- (void)longPress {
    CABasicAnimation *animation = (CABasicAnimation *)[imageView.layer animationForKey:@"rotation"];
    if (animation == nil) {
        [self shakeImage];
    }else {
        [self resume];
    }
}
```

```
//如果点击图标外区域，停止抖动
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event {
    UITouch *touch = [touches anyObject];
    CGPoint point = [touch locationInView:self.view];
    //转换坐标系，判断touch点是否在imageView内，在的话，仍然抖动，否则停止抖动
    CGPoint p = [self.view convertPoint:point toView:imageView];
    if (![imageView pointInside:p withEvent:event]) {
        NSLog(@"xxxxxxx");
        [self pause];
    }
}

//layer.speed
/* The rate of the layer. Used to scale parent time to local time, e.g.
 * if rate is 2, local time progresses twice as fast as parent time.
 * Defaults to 1. */
//这个参数的理解比较复杂，我的理解是所在layer的时间与父layer的时间的相对速度，为1时两者速度一样，为2那么父layer过了一秒，而所在layer过了两秒（进行两秒动画）,为0则静止。
- (void)pause {
    imageView.layer.speed = 0.0;
}

- (void)resume {
    imageView.layer.speed = 1.0;
}
```


3) 动画设置
```
- (void)shakeImage {
    //创建动画对象,绕Z轴旋转
    CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:@"transform.rotation.z"];

    //设置属性，周期时长
    [animation setDuration:0.08];

    //抖动角度
    animation.fromValue = @(-M_1_PI/2);
    animation.toValue = @(M_1_PI/2);
    //重复次数，无限大
    animation.repeatCount = HUGE_VAL;
    //恢复原样
    animation.autoreverses = YES;
    //锚点设置为图片中心，绕中心抖动
    imageView.layer.anchorPoint = CGPointMake(0.5, 0.5);

    [imageView.layer addAnimation:animation forKey:@"rotation"];
}
```
