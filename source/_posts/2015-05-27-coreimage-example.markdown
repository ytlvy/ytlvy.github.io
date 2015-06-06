---
layout: post
title: "Core Image使用"
date: 2015-05-27 15:52:52 +0800
comments: true
categories: IOS
---
首先创建一个径向渐变的滤镜，该滤镜是从白到黑的渐变方式，白色区域的半径默认是100。接着将其与一张使用CIDarkenBlendMode滤镜的图片合成。CIDarkenBlendMode的作用是背景图片样本将被源图片的黑色部分替换掉。

```
UIImage* moi = [UIImage imageNamed:@"Mars.jpeg"];

CIImage* moi2 = [[CIImage alloc] initWithCGImage:moi.CGImage];

CIFilter* grad = [CIFilter filterWithName:@"CIRadialGradient"];

CIVector* center = [CIVector vectorWithX:moi.size.width / 2.0 Y:moi.size.height / 2.0];

// 使用setValue：forKey：方法设置滤镜属性
[grad setValue:center forKey:@"inputCenter"];

// 在指定滤镜名时提供所有滤镜键值对
CIFilter* dark = [CIFilter filterWithName:@"CIDarkenBlendMode" keysAndValues:@"inputImage", grad.outputImage, @"inputBackgroundImage", moi2, nil];

CIContext* c = [CIContext contextWithOptions:nil];

CGImageRef moi3 = [c createCGImage:dark.outputImage fromRect:moi2.extent];

UIImage* moi4 = [UIImage imageWithCGImage:moi3 scale:moi.scale orientation:moi.imageOrientation];

CGImageRelease(moi3);
```

![图片合成图](http://images.cnitblog.com/blog/429321/201212/27103647-2fcd19d1ca1e4a0d9f42e2c79687e881.png)

