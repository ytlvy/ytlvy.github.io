title: 'AutoLayout -- Content Compression Resistance & Content Hugging'
categories: IOS
tags:
  - autolayout
description:
date: 2015-06-08 21:26:29
author:
---

## Content Compression Resistance && Content Hugging 
Auto Layout中, 存在Content Compression Resistance 和 Content Hugging 这两个概念.这两个概念是在`固有内容尺寸`（Intrinsic Content Size）之上起作用的.


### Intrinsic Content Size
包含内容的UI控件, 由内容多少而决定的大小规则.

### Content Compression Resistance & Content Hugging
1. 内容大小改变是指, 内容显示内容所占空间的长度来说的.
2. `内容抗压指数`，在父视图变小时, 会根据抗压指数来缩小各子控件;
3. `内容拥抱指数`, 内容越集中于控件中心, 周围空白越小.

<!-- more -->

### 例子
假设，你有一个下面这样的按钮：
```
[       Click Me        ]
```
按钮与其父视图之间的边距约束优先级是500。
然后，如果按钮的吸附性优先级（Hugging priority）大于500，按钮看起来会是这样：

```
[Click Me]
```

如果，吸附性优先级小于500，按钮会是这样：
```
[       Click Me        ]
```

然后，如果现在父视图收缩了，按钮的压缩阻力优先级（Compression Resistance priority）大于500，它看起来会是这样：
```
[Click Me]
```

否则，如果压缩阻力优先级小于500，它会是这样：
```
[Cli..]
```

如果不是这样，则很可能是有一些其他的约束扰乱了你的整个布局！ 例如，可能你的边距约束优先级是1000。或者你可能有一个优先级较高的宽度约束。如果遇到这种情况，可以试试“Editor > Size to Fit Content”菜单命令。








