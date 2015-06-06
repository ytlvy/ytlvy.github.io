---
layout: post
title: "ios BOOL / bool /   Boolean / NSCFBoolean"
date: 2015-06-05 16:14:34 +0800
comments: true
categories: IOS
---
## BOOL  bool   Boolean  NSCFBoolean
### BOOL
> Objective-C defines BOOL to encode truth value. It is a typedef of a signed char, with the macros YES and NO to represent true and false, respectively.
> On a 64-bit iOS, BOOL is defined as a bool, rather than signed char, which precludes the runtime from these type conversion errors.
> 0 is considered "false", while any other value is considered "true". Because NULL and nil have a value of 0, they are considered "false" as well.

<!--more-->

### The Wrong Answer to the Wrong Question

```
static BOOL different (int a, int b) {
    return a - b;
}
```

However, because BOOL is typedef 'd as a signed char on 32- bit architectures, this will not behave as expected:

```
different(11, 10) // YES
different(10, 11) // NO (!)
different(512, 256) // NO (!)
```

***Deriving truth value directly from an arithmetic operation is never a good idea***. Use the == operator, or cast values into booleans with the ! (or !!) operator.

### The Truth About NSNumber and BOOL

```
 NSLog(@"%@", [@(YES) class]);
 //__NSCFBoolean
```

Name      |Type                 |Header       |True     |False   
----------|---------------------|-------------|---------|--------
BOOL      |signed char / bool   |objc.h       |YES      |NO      
bool      |_Bool (int)          |stdbool.h    |TRUE     |FALSE   
Boolean   |unsigned char        |MacTypes.h   |TRUE     |FALSE   
NSNumber  |__NSCFBoolean        |Foundation.h |@(YES)   |@(NO)  
