---
layout: post
title: "ios static formatter"
date: 2015-06-05 09:47:14 +0800
comments: true
categories: IOS
---
### Re-Using Formatter Instances
```
- (void)fooWithNumber: (NSNumber *)num{
  static NSNumberFormatter *_formatter = nil;
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    _formatter = [[NSNumberFormatter alloc] init];
    [_formatter setNumberStyle: NSNumberFormatterDecimalStyle];
    });

    NSString *string = [_formatter stringFromNumber: number];
}

```

<!--more-->

```
+ (NSNumberFormatter *)numberFormatter {
    static NSNumberFormatter *_formatter = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _formatter = [[NSNumberFormatter alloc] init];
        [_formatter setNumberStyle:
          NSNumberFormatterDecimalStyle];
    });
    return _formatter;
}
```
