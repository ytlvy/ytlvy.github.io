title: ios GPUImage simple use
categories: IOS
tags:
  - Vendor
date: 2015-08-04 21:26:07
author:
description:
photos:
---
## GPUImage
GPUImage 作者是 Brad Larson，以 BSD 协议放出，能够在图像、实时摄像头影像和视频上使用 GPU 加速的滤镜和其他效果.

### GPU vs. CPU
每只 iPhone 都有两个处理器：一个 CPU，也就是中央处理器，和一个 GPU，也就是图形处理器。每个处理器都有它自己的优势，现代芯片架构（例如 Apple A4）把 CPU 和 GPU 集成在一个物理封装里。

在 Xcode 里面写 C 和 Objective-C 的时候，产生的指令绝大部分会被 CPU 执行。相对地，GPU 是一枚专用芯片，专门用来做独立的小操作，例如图形渲染。GPU 执行的指令和 CPU 是有很大区别的，因此用特殊的语言来编写：OpenGL（特别地，在 iPhone 和 iPad 上使用 OpenGL ES）

### 渲染管线
![](http://nshipster.s3.amazonaws.com/gpuimage-pipeline.png)

GPUImage 从本质上来说是一个渲染管线的 `Objective-C `抽象。从相机、网络或者磁盘上加载的源图像，在经过一系列滤镜处理后，最终输出到了 `view`、`graphics context` 或是数据流中。

例如说，摄像头中的图像可以应用一个 Color Levels 滤镜，来模拟各种色盲效果，然后实时显示在一个 view 中。
```
GPUImageVideoCamera *videoCamera = [[GPUImageVideoCamera alloc]
    initWithSessionPreset:AVCaptureSessionPreset640x480
           cameraPosition:AVCaptureDevicePositionBack];
videoCamera.outputImageOrientation = UIInterfaceOrientationPortrait;

GPUImageFilter *filter = [[GPUImageLevelsFilter alloc] 
                            initWithFragmentShaderFromFile:@"CustomShader"];
[filter setRedMin:0.299 gamma:1.0 max:1.0 minOut:0.0 maxOut:1.0];
[filter setGreenMin:0.587 gamma:1.0 max:1.0 minOut:0.0 maxOut:1.0];
[filter setBlueMin:0.114 gamma:1.0 max:1.0 minOut:0.0 maxOut:1.0];
[videoCamera addTarget:filter];

GPUImageView *filteredVideoView = [[GPUImageView alloc] initWithFrame:self.view.bounds)];
[filter addTarget:filteredVideoView];
[self.view addSubView:filteredVideoView];

[videoCamera startCameraCapture];
```

或者，结合不同的颜色混合模式，图像效果和一些调整，你可以把静止图像转变成可以分享给你时髦的朋友们看看的精美图像（例子来自基于 GPUImage 的 FilterKit）。
```
GPUImageFilterGroup *filter = [[FKGPUFilterGroup alloc] init];

GPUImageSaturationFilter *saturationFilter = [[GPUImageSaturationFilter alloc] init];
[saturationFilter setSaturation:0.5];

GPUImageMonochromeFilter *monochromeFilter = [[GPUImageMonochromeFilter alloc] init];
[monochromeFilter setColor:(GPUVector4){0.0f, 0.0f, 1.0f, 1.0f}];
[monochromeFilter setIntensity:0.2];

GPUImageVignetteFilter *vignetteFilter = [[GPUImageVignetteFilter alloc] init];
[vignetteFilter setVignetteEnd:0.7];

GPUImageExposureFilter *exposureFilter = [[GPUImageExposureFilter alloc] init];
[exposureFilter setExposure:0.3];

[filter addGPUFilter:exposureFilter];
[filter addGPUFilter:monochromeFilter];
[filter addGPUFilter:saturationFilter];
[filter addGPUFilter:vignetteFilter];
```

看完了 GPUImage 的功能，一定很兴奋吧。GPUImage 非常简单，可以马上开始使用（不需要了解 OpenGL），性能上也足够完成你想要的一切。不止是这样，它还自带了非常多的积木一样的部件 -- 色彩调整，混合模式，你能想到的想不到的一切视觉效果。

GPUImage 是开源社区少有的好东西，我们 Mac 和 iOS 开发者非常幸运的能拥有它。用 GPUImage 做些牛逼的东西，向其他人展示一个崭新的世界

### 吐槽
我自己也用过一点 GPUImage，方便是很方便，不光是省掉了 AVFoundation 的许多烦人的 setup 代码，还内置了许多滤镜效果。不过它也有不少坑，我遇到的一个就是占用太多内存，一段时间后就 crash 掉了，毕竟是处理视频，需要不少内存，iPhone 的内存又不太多，就要更加注意。我当时是加了一个 @autoreleasepool 就好多了。
