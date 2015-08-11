title: iOS shake in pull to refresh
categories: IOS
tags:
  - Optimization
  - UIScrollView

date: 2015-08-08 17:28:14
author:
---
[reference](http://blog.cnbluebox.com/blog/2015/01/19/shake-in-pull-to-refresh/)
## 你的下拉刷新是否“抖”了一下

在进入IOS8之后，你有没有注意到老式的下拉刷新可能会抖一下， 在下拉松开后，scrollView即将回到“刷新中…”的状态过程中的时候。如果你又这个问题，那不妨跟随我来看看怎么解决这个问题。

### 抖动的原因
我们先来看看在手松开之后我们对scrollView做了什么事情：

`ScrollViewDidEndDragging` => `setContentInset:`

为了保证在“Loading”的状态下，下拉刷新控件可以展示，我们对`contentInset`做了修改，增加了`inset`的`top`. 那这样一步操作为什么会导致scrollView抖动一下呢。

我在`scrollViewDidScroll:`中打了个断点，来看看在`setContentInset:`之后发生了什么事情。 我设置的`inset.top = 64`; 结果发现scrollView的`contentOffset`发生了这样的变化：

```
(0, -64) => (0, -133) => (0, -64)
```

由以上数据可以看出，contentOffset在这个过程中先被向下移动了一段，再回归正常。 猜测问题原因：
```
下拉松开之后， scrollView本身的 bounce 效果 与 当前设置inset冲突了
```

<!-- more -->

### 初步尝试： async
知道问题的原因后，我第一思路是避开这个冲突，于是我把 setContentInset: 的方法异步调用一下：
```
dispatch_async(dispatch_get_main_queue(), ^{
    [UIView animateWithDuration:kAnimationDuration animations:^{
        self.scrollView.contentInset = inset;
    } completion:^(BOOL finished) {
    }];
});
```

尝试了一下，问题没有了。然而有人跟我说还是遇到了这个问题, 经过验证，确实这个问题还是没有被完全修复。

### 二次修改： 强设contentOffset
既然是因为contentOffset改变导致的，我就再设置一下contentOffset应该就行了。于是二次尝试：
```
dispatch_async(dispatch_get_main_queue(), ^{
    [UIView animateWithDuration:kAnimationDuration animations:^{
        self.scrollView.contentInset = inset;
        self.scrollView.contentOffset = CGPointMake(0, -inset.top);
    } completion:^(BOOL finished) {
    }];
});
```

试验结果发现，没用，问题还是存在，在这一步耗了不少时间想尽其他办法都没搞定问题，直到我将 setContentOffset: 方法改为 `setConentOffset:animated:` 。 问题就解决了！看来系统里面这两个方法的实现是不同的啊。

```
dispatch_async(dispatch_get_main_queue(), ^{
    [UIView animateWithDuration:kAnimationDuration animations:^{
        self.scrollView.contentInset = inset;
        [self.scrollView setContentOffset:CGPointMake(0, -inset.top) animated:NO];
    } completion:^(BOOL finished) {
    }];
});
```

