---
layout: post
title: "UITextView &amp;&amp; UITextField"
date: 2015-05-27 15:48:05 +0800
comments: true
categories: IOS
---
### UITextView
#### ios7 textView contentInsets
    A text view is a scroll view. iOS 7 will add a content offset automatically to scroll views, as it assumes they will want to scroll up behind the nav bar and title bar.
    To prevent this, override the new method on UIViewController, automaticallyAdjustsScrollViewInsets, and return NO.

### UITextField
#### textfield clear button bug
    #pragma mark- gesture delegate
    - (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldReceiveTouch:(UITouch *)touch {
        if ([touch.view isKindOfClass:[UITextField class]] ||
            [touch.view isKindOfClass:[UIButton class]])
        {
            return NO;
        }
        return YES;
    }
