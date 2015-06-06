---
layout: post
title: "ios 知识点"
date: 2015-05-27 15:46:55 +0800
comments: true
categories: IOS
---
####  正确设定背景图片
1. 创建小的重复的图片作为背景的, 使用UIColor的 colorWithPatternImage来设置背景色.
2. 大背景图, 在view中添加一个UIImageView作为一个子View

#### 设定Shadow Path
    view.layer.shadowPath = [[UIBezierPath bezierPathWithRect:view.bounds] CGPath];

#### 优化Table View
正确使用`reuseIdentifier`来重用cells
尽量使所有的view opaque，包括cell自身
避免渐变，图片缩放，后台选人
缓存行高
如果cell内现实的内容来自web，使用异步加载，缓存请求结果
使用`shadowPath`来画阴影
减少subviews的数量
尽量不适用`cellForRowAtIndexPath:`，如果你需要用到它，只用一次然后缓存结果
使用正确的数据结构来存储数据
使用`rowHeight`, `sectionFooterHeight` 和 `sectionHeaderHeight`来设定固定的高，不要请求delegate

#### 选择是否缓存图片
imageWithContentsOfFile 不缓存
imageNamed 缓存

#### block array need copy first

#### 查看引用计数
    NSLog(@"retain count = %d", _objc_rootRetainCount(obj));
#### 将objc文件编译成C++文件
    clang -rewrite-objc file_name_of_the_source_code


#### nonatomic && atomic
nonatomic属性读取的是内存数据（寄存器计算好的结果）
atomic就保证直接读取寄存器的数据

#### set do not back up attribute
    NSURL * fileURL;
    fileURL = [ NSURL fileURLWithPath: @"some/file/path" ];
    [ fileURL setResourceValue: [NSNumber numberWithBool: YES] forKey: NSURLIsExcludedFromBackupKey error: nil ];