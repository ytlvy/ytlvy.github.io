---
layout: post
title: "ios 自动布局"
date: 2015-05-27 15:49:30 +0800
comments: true
categories: IOS
---
### 自动布局
#### IOS7 UIlabel 高度bug
    //需要设置
    lb1.lineBreakMode = NSLineBreakByCharWrapping;

#### 自动布局bug
    使用autolayout时 button不能点击 可能是因为button的parentView 没有自动扩展开

#### 系统自带autolayout
    viewsDictionary = NSDictionaryOfVariableBindings(scrollView, imageView);
    [self.view addConstraints:[NSLayoutConstraint constraintsWithVisualFormat:@"H:|[scrollView]|" 
                                                options:0 metrics: 0 views:viewsDictionary]];
    [self.view addConstraints:[NSLayoutConstraint constraintsWithVisualFormat:@"V:|[scrollView]|" 
                                                options:0 metrics: 0 views:viewsDictionary]];
    [scrollView addConstraints:[NSLayoutConstraint constraintsWithVisualFormat:@"H:|[imageView]|" 
                                                options:0 metrics: 0 views:viewsDictionary]];
    [scrollView addConstraints:[NSLayoutConstraint constraintsWithVisualFormat:@"V:|[imageView]|" 
                                                options:0 metrics: 0 views:viewsDictionary]];

#### AutoLayout priority
    [UIView autoSetPriority:UILayoutPriorityRequired forConstraints:^{
        [self.titleLabel autoSetContentCompressionResistancePriorityForAxis:ALAxisVertical];
    }];
    [self.titleLabel autoPinEdgeToSuperviewEdge:ALEdgeTop withInset:kLabelVerticalInsets];
    [self.titleLabel autoPinEdgeToSuperviewEdge:ALEdgeLeading withInset:kLabelHorizontalInsets];
    [self.titleLabel autoPinEdgeToSuperviewEdge:ALEdgeTrailing withInset:kLabelHorizontalInsets];