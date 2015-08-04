title: iOS bit computing
categories: IOS
tags:
  - Bit
date: 2015-08-04 21:30:00
author:
description:
photos:
---
## iOS 位与位运算

### 介绍
位操作是程序设计中对位模式或二进制数的一元和二元操作. 在许多古老的微处理器上, 位运算比加减运算略快, 通常位运算比乘除法运算要快很多. 在现代架构中, 情况并非如此:位运算的运算速度通常与加法运算相同(仍然快于乘法运算).（摘自wikipedia）
OC作为c的扩展和超集，位运算自然使用的是c的操作符。c提供了6个位操作符，$，|，^，~，<<，>>。本文不打算做位运算的基础教学，只介绍一些开发中能用到的场景。

### 提高运算速度
如前一段所说，位运算的运算速度是通常与加法速度相当，但是快于乘法运算的。故而如果我们的程序对性能有要求，我们可以使用位运算来提高运算速度。比如：

- 乘以2：n << 1;
- 除以2：n >> 1;
- 乘以2的m次方：n << m;
- 除以2的m次方：n >> m;
- 判断奇偶：(n & 1) == 1;
- 求平均数：(a + b) >> 1;
- ……

基于乘除法的位运算提速还有很多，这里不一一列举。需要注意的是，你应当只在遇到性能瓶颈的时候，并且瓶颈的确是计算的时候才这么做。因为使用位运算并不利于程序的可读性和可维护性。（科学计算除外）

### 压缩空间
以前接触过ACM的筒子们应该对状态压缩不陌生，状态压缩的目的在于把一个大数据用有限的内存空间来进行表示。比如 Programming Pearls 里面的一个经典示例：如何对最多有一千万条不重复的7位整数（电话号码）进行排序？且可使用的内存空间有大约1MB多。

显而易见的常规做法既是做一个基于磁盘操作的外排序。然而如果转换一下思路，充分的使用内存中的每一个位，加上不存在重复的电话号码，以及不存在0和1开头的电话号码。我们只需要使用1000万个位（大约1.2mb），就能以集合的方式在内存里标记下所有的数据，从而轻松的实现位排序。此种方法大幅度的减少了IO时间，从而获得巨大的性能提升。

ACM里面有大量的如果使用位来压缩空间的示例，状态压缩的动态规划等，此处不做展开，只告诉读者，充分的使用内存的每一个位，经常能带来意想不到的收获。但需要注意的是，状态的压缩和提取，都需要一定的计算量，有时一味的追求状态压缩，反而会降低效率。

### 表示数据
比较经典的一个应用场景，使用一串24位的十六机制数字来表现一个RGB颜色（或者32位来表示ARGB）。由于PS，Web以及各类取色器，都能快速的取出RGB的Hex值，但是UIColor没有对应的方法。故而我们可以写出下面这样一个UIColor的Category，来快速的用一个RGBHex生成一个UIColor。（源码在[UIColor + CYHelper.h](http://github.com/lancy/cyhelper)）
```
+ (UIColor *)colorWithRGBHex:(UInt32)hex
{
    return [UIColor colorWithRGBHex:hex alpha:1.0f];
}

+ (UIColor *)colorWithRGBHex:(UInt32)hex alpha:(CGFloat)alpha
{
    int r = (hex >> 16) & 0xFF;
    int g = (hex >> 8) & 0xFF;
    int b = (hex) & 0xFF;

    return [UIColor colorWithRed:r / 255.0f green:g / 255.0f blue:b / 255.0f alpha:alpha];
}
```

状态与选项
```
typedef NS_OPTIONS(NSUInteger, UIViewAnimationOptions) {
    UIViewAnimationOptionLayoutSubviews            = 1 <<  0,
    UIViewAnimationOptionAllowUserInteraction      = 1 <<  1, // turn on user interaction while animating
    UIViewAnimationOptionBeginFromCurrentState     = 1 <<  2, // start all views from current value, not initial value
    UIViewAnimationOptionRepeat                    = 1 <<  3, // repeat animation indefinitely
    UIViewAnimationOptionAutoreverse               = 1 <<  4, // if repeat, run animation back and forth
    UIViewAnimationOptionOverrideInheritedDuration = 1 <<  5, // ignore nested duration
    UIViewAnimationOptionOverrideInheritedCurve    = 1 <<  6, // ignore nested curve
    UIViewAnimationOptionAllowAnimatedContent      = 1 <<  7, // animate contents (applies to transitions only)
    UIViewAnimationOptionShowHideTransitionViews   = 1 <<  8, // flip to/from hidden state instead of adding/removing
    UIViewAnimationOptionOverrideInheritedOptions  = 1 <<  9, // do not inherit any options or animation type

    UIViewAnimationOptionCurveEaseInOut            = 0 << 16, // default
    UIViewAnimationOptionCurveEaseIn               = 1 << 16,
    UIViewAnimationOptionCurveEaseOut              = 2 << 16,
    UIViewAnimationOptionCurveLinear               = 3 << 16,

    UIViewAnimationOptionTransitionNone            = 0 << 20, // default
    UIViewAnimationOptionTransitionFlipFromLeft    = 1 << 20,
    UIViewAnimationOptionTransitionFlipFromRight   = 2 << 20,
    UIViewAnimationOptionTransitionCurlUp          = 3 << 20,
    UIViewAnimationOptionTransitionCurlDown        = 4 << 20,
    UIViewAnimationOptionTransitionCrossDissolve   = 5 << 20,
    UIViewAnimationOptionTransitionFlipFromTop     = 6 << 20,
    UIViewAnimationOptionTransitionFlipFromBottom  = 7 << 20,
} NS_ENUM_AVAILABLE_IOS(4_0);
```

我们观察Apple在`UIViewAnimationOptions`的枚举变量，使用了一个NSUInteger就表示了`UIViewAnimation`所需的所有Option。其中0~9十个是互不影响的可同时存在option。16~19，20~24使用了4位来表示互斥的option。

如此定义了之后，对`UIViewAnimationOptions`的赋值变得尤为简单，使用 | 操作符既可以获得一个给对应的option位赋值后的结果。例如：
```
[UIView animateWithDuration:1.0
                      delay:0
                    options:UIViewAnimationOptionAllowUserInteraction
                         | UIViewAnimationOptionBeginFromCurrentState
                         | UIViewAnimationOptionCurveEaseIn
                 animations:{...}
                 completion:{...}];
```

提取也比较简单，使用 `&` 操作符 和 `>>` 操作符，就可以轻松判定某个位有没有被设置，以及提取某些状态位，例如：
```
UIViewAnimationOptions option = UIViewAnimationOptionAllowUserInteraction
                                | UIViewAnimationOptionBeginFromCurrentState
                                | UIViewAnimationOptionCurveEaseIn
                                | UIViewAnimationOptionTransitionCrossDissolve;

if (option & UIViewAnimationOptionAllowUserInteraction) {
    NSLog(@"UIViewAnimationOptionAllowUserInteraction has been set");
}
if (option & UIViewAnimationOptionBeginFromCurrentState) {
    NSLog(@"UIViewAnimationOptionBeginFromCurrentState has been set");
}
UInt8 optionCurve = option >> 16 & 0xf;
if (optionCurve == 1) {
    NSLog(@"UIViewAnimationOptionCurveEaseIn has been set");
}
UInt8 optionTransition = option >> 20 & 0xf;
if (optionTransition == 5) {
    NSLog(@"UIViewAnimationOptionTransitionCrossDissolve has been set");
}
```

这里最需要注意的地方就是，对互斥的状态的设置必须尤为小心，如果你这么写：
```
UIViewAnimationOptions badOption = UIViewAnimationOptionCurveEaseIn | UIViewAnimationOptionCurveEaseOut;
UInt8 oops = badOption >> 16 & 0xf;
NSLog(@"Sorry, it's not UIViewAnimationOptionCurveEaseInOut");
NSLog(@"oops = %d, you got UIViewAnimationOptionCurveLinear", oops);
```

