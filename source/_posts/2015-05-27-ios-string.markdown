---
layout: post
title: "ios 字符串处理"
date: 2015-05-27 15:49:39 +0800
comments: true
categories: IOS
---
### 字符串

#### 字符创建区别
```
    @"sss" 创建在常量区
    [NSString stringWithFormat:@"Hello world"] 创建在堆上
```

#### 字符串比较
```
    enum {
       NSOrderedAscending = -1,
       NSOrderedSame,
       NSOrderedDescending
    };
    typedef NSInteger NSComparisonResult;
    NSString caseInsensitiveCompare:@"Response" == NSOrderedSame
```

<!--more-->

#### 字符串以什么开头
```
    NSURL *url; [[url absoluteString] hasPrefix:@""]
```

#### insert text at cursor
```
    [textView replaceRange:textView.selectedTextRange withText:insertingString];
```

#### NSData to NSString
```
    NSString* newStr = [[NSString alloc] initWithData:theData encoding:NSUTF8StringEncoding];
    NSString* newStr = [NSString stringWithUTF8String:[theData bytes]]
    let newStr = NSString(data: data, encoding: NSUTF8StringEncoding)
```

#### 文本段落
```
    NSMutableParagraphStyle *style = [[NSMutableParagraphStyle alloc] init];
    style.firstLineHeadIndent = 50;
    [attributedText addAttribute:NSParagraphStyleAttributeName value:style range:range];
    [valueLabel setAttributedText:attributedText];
```

#### 删除空白
```
    //trim blank
    NSString *cleanString = [dirtyString stringByTrimmingCharactersInSet:
                [NSCharacterSet whitespaceAndNewlineCharacterSet]];
    // trim content blank
    NSString *theString = @"    Hello      this  is a   long       string!   ";
    NSCharacterSet *whitespaces = [NSCharacterSet whitespaceCharacterSet];
    NSPredicate *noEmptyStrings = [NSPredicate predicateWithFormat:@"SELF != ''"];
    NSArray *parts = [theString componentsSeparatedByCharactersInSet:whitespaces];
    NSArray *filteredArray = [parts filteredArrayUsingPredicate:noEmptyStrings];
    theString = [filteredArray componentsJoinedByString:@" "];
```

#### 字符串编码转换
```
    GB2312 NSStingEncoding enc = CFStringConvertEncodingToNSStringEncoding(kCFStringEncodingGB_18030_2000);

    NSError *err = nil;
    NSString *urlContent = [NSString stringWithContentOfURL:url, encoding:enc, error:&err];
    if(err){
        NSLog(@"error=%@", [err description]);
    }
```

#### 字符串转义
```
    NSString *str = [@"http://www.163.com", stringByAddingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
    NSString *str = [str1 stringByReplacePercentEscapesUsingEncoding:NSUTF8StringEncoding];
```

