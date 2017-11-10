---
author: null
categories:
- IOS
date: '2015-08-08'
tags:
- Autolayout
title: iOS Autolayout Practices
---

[reference](http://blog.cnbluebox.com/blog/2015/02/02/autolayout2/)

## Autolayout小结

###  如何自动适应cell的高度

在IOS的布局中，计算和适应cell的高度是个经典的问题, 在frame时代，我们都知道用`sizeWithFont:` 先计算出文字的高度，然后通过计算得出cell的高度，然后赋予`heightForRow:`。

那在Autolayout时代如何计算cell的高度呢？因为sizeWithFont:方法已经不太实用了。其实Autolayout不但更简单，还可以不用写过多的计算代码达到自适应高度。

理论上是可以通过已知的完整的Constraints和view的属性来计算高度的，我们可以通过`systemLayoutSizeFittingSize:`方法来获取计算出来cell的size，我们知道cell的高度需要在tableView的代理方法`tableView:heightForRowAtIndexPath:`中实现的，那么我们考虑从以下两点来做：

1. 通过创建一个额外的cell专门用来计算其高度
2. 因为计算需要布局，所以尽量让其只计算一次，计算完可以将高度保存起来

<!--more-->

基于这两个要点，我们可以尝试如下(伪代码)：
```
- (CGFloat)autoAdjustedCellHeightAtIndexPath:(NSIndexPath *)indexPath inTableView:(UITableView *)tableView {

    CGFloat cellHeight = [self cellHeightAtIndexPath:indexPath];
    if (cellHeight > 0) {
        return cellHeight;
    } else {
      //此方法创建用于自适应的cell, 注意创建一个就够了
        UITableViewCell *cell = [self cellForTableViewAutoAdjust];

        //更新cell,同cellForRow中更新一致
        [self updateCell:cell]

        //让cell进行layout
        [cell setNeedsUpdateConstraints];
        [cell updateConstraintsIfNeeded];
        cell = CGRectMake(0.0f, 0.0f, CGRectGetWidth(tableView.bounds), CGRectGetHeight(layoutGuideView.bounds));
        [cell setNeedsLayout];
        [cell layoutIfNeeded];

        CGFloat height = [cell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize].height;

        height += 1.0f;
        //计算完后保存，避免多次重复计算
        [self saveCellHeight:height forIndexPath:indexPath];
        return height;
    }
}
```
经过测试，工作良好，在体会到这点好处之后就发现Autolayout的好处了，它其实是让布局变得简单了。
在完成这个自动适应高度的设计过程中要注意几点：

1. cell中的多行UILabel，如果其width不是固定的话（比如屏幕尺寸不同，width不同），要手动设置其`preferredMaxLayoutWidth`。 因为计算UILabel的`intrinsicContentSize`需要预先确定其width才行。
2. 要保证垂直方向上的`Constraints`理论上可以计算出cell的高度，另：最好调低其中一个Constraint的优先级，避免约束发生冲突
3. tableView的`dequeueReusableCellWithIdentifier`取出来的cell如果未用于tableView, 那么它就leak了. 因此用于计算高度的cell不要从此方法获取。

### 如何在scrollView中使用Autolayout
crollView比较特殊，因为它有个contentSize的属性。那么在遇到scrollView时，怎么使用Autolayout呢。其实关键点就一点：
***ScrollView的contentSize的大小是由其subview的constraints来决定的。***

知道这一点其实就够了，那么基于这个特性，在应用的时候要注意哪些呢？

1. 完全依赖scrollView来计算subview的坐标的Constraints设法不行了，计算条件互相依赖的，常见的如left+right+top+bottom不行了。
2. 要保证contentSize可以通过subview的constraints能够计算出来。

举个栗子：
```
[childView mas_makeConstraints:^(MASConstraintMaker *make) {
    make.width.mas_equalTo(scrollView.mas_width);
  make.height.mas_equalTo(scrollView.mas_height);
  make.leading.mas_equalTo(scrollView.mas_leading);
  make.top.mas_equalTo(scrollView.mas_top);
  make.bottom.mas_equalTo(scrollView.mas_bottom);
  make.trailing.mas_equalTo(scrollView.mas_trailing).multipliedBy(1./4.);
}];
```

例子中如果scrollView的size是(100x100), 那么contentSize即为(400x100)

### Autolayout时的动画
在使用Autolayout时，动画的使用和以前也不同了，以前我们是修改frame，现在我们可以通过修改Constraints, 然后在动画时`layoutIfNeeded`就行了。

```
[UIView animateWithDuration:0.2 animations:^{
    [view layoutIfNeeded];
}];
```
Autolayout有时在动画时候会很方便，因为View之间的坐标是相互影响的，在传统frame中，如果改变一个view的frame,那么可能你要更改很多view的frame，才能让页面显得和谐。在Autolayout中可能只需要修改一个Constraint就可以了，在做动画时会很方便。

### Autolayout在IOS6上的坑
虽然Autolayout非常强大，但是在刚出现的版本IOS6上，还是有一些坑的。在IOS6上你会发现在某些系统控件上如： UITableViewCell, UITableView 等view上直接添加Constraints会crash。 冲突的Constraints会直接导致crash等。
在strackoverflow有回答上说明了为什么UITableViewCell等部分系统控件上添加Constraints会crash, 即UITableViewCell等部分控件的layoutSubviews里面没有执行`[super layoutSubviews]`。
所以想要使用Autolayout的同学们，如果你要兼容IOS6的话，有些问题还是要重视的，需要充分的测试哦。





