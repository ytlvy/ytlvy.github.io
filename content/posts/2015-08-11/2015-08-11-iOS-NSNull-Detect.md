---
categories:
- IOS
date: '2015-08-11'
tags:
- NSNULL
title: iOS NSNull Detect
---

## Object-C中nil, NULL跟NSNull

相信不少开发者，都被NSNull坑过，最常见的是服务器返回的json里面，说好的字典、数组、数字，结果返回的是空值。这个时候，NSJSONSerialization 会自动把他们换成 NSNull。当我们再去用dict[@“hello”]的时候，就会出触发exception，导致程序崩溃。

### 最简单的做法
相信大家都知道，[NSNull null] 并不是一个工厂方法，而是一个单例模式，那么我们直接去判断赋值的这个指针是不是[NSNull null] 就好了。

那么问题来了，编译器会多了一个warning，很烦人。

在[这篇文章](http://www.takingnotes.co/blog/2012/01/06/comparing-to-nsnull/)里面介绍了各种做法：

```
- (void)someMethod 
{ 
    NSString *aString = @"loremipsum"; 
 
    // This will complain: "Comparison of distinct pointer types ('NSString *' and 'NSNull *')" 
    if (aString != [NSNull null]) 
    { 
 
    } 
 
    // This works (at least for strings), but isEqual: does different things  
    // for different classes, so it's not ideal 
    if ([aString isEqual:[NSNull null]]) 
    { 
 
    } 
 
    // If you cast it to the class you're comparing against 
    // then you're good to go 
    if (aString != (NSString *)[NSNull null]) 
    { 
 
    } 
 
    // But we can also just cast it to id and 
    // that works generically 
    if (aString != (id)[NSNull null]) 
    { 
 
    } 
 
    // The thing that would be really cool, 
    // would be [NSNull null] returning 
    // id (like in the sample category below). 
    // Wouldn't count on that one though. 
    if (aString != [NSNull idNull]) 
    { 
 
    } 
} 
```

这些都不是非常漂亮的解决方案，这篇文章的作者推荐：
```
@interface NSNull (idNull) 
+ (id)idNull; 
@end 

@implementation NSNull (idNull) 
+ (id)idNull { return [NSNull null]; } 
@end 
```

或者呢
```
if ([[NSNull null] isEqual:aString]) 
{ 
 
} 
```

### 最终解决方案

上面的做法，都需要判断一次，还是很不优雅，为什么呢，我们还是不能像NULL，nil一样，直接拿来用，还是需要判断一下，这里推荐一套最漂亮的作法。

陈航提供了一个[gist](https://gist.github.com/cyndibaby905/9828745)
```
#define NSNullObjects @[@"",@0,@{},@[]] 
 
@interface NSNull (InternalNullExtention) 
@end 
 
@implementation NSNull (InternalNullExtention) 
 
- (NSMethodSignature*)methodSignatureForSelector:(SEL)selector 
{ 
    NSMethodSignature* signature = [super methodSignatureForSelector:selector]; 
    if (!signature) { 
        for (NSObject *object in NSNullObjects) { 
            signature = [object methodSignatureForSelector:selector]; 
            if (signature) { 
                break; 
            } 
        } 
 
    } 
    return signature; 
} 
 
- (void)forwardInvocation:(NSInvocation *)anInvocation 
{ 
    SEL aSelector = [anInvocation selector]; 
 
    for (NSObject *object in NSNullObjects) { 
        if ([object respondsToSelector:aSelector]) { 
            [anInvocation invokeWithTarget:object]; 
            return; 
        } 
    } 
 
    [self doesNotRecognizeSelector:aSelector]; 
} 
@end 
```

这里还提供一个日本人的封装方案：
```
#import "NSNull+OVNatural.h" 
 
@implementation NSNull (OVNatural) 
- (void)forwardInvocation:(NSInvocation *)invocation 
{ 
    if ([self respondsToSelector:[invocation selector]]) { 
        [invocation invokeWithTarget:self]; 
    } 
} 
 
- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector 
{ 
    NSMethodSignature *sig = [[NSNull class] instanceMethodSignatureForSelector:selector]; 
    if(sig == nil) { 
        sig = [NSMethodSignature signatureWithObjCTypes:"@^v^c"]; 
    } 
    return sig; 
} 
 
@end 
```

### nil 判定

```
//判断对象不空
if(object) {}

//判断对象为空
if(object == nil) {}

//数组初始化，空值结束
NSArray *pageNames=[[NSArray alloc] initWithObjects:@"DocumentList",@"AdvancedSearch",@"Statistics",nil];

//判断数组元素是否为空
UIViewController *controller=[NSArray objectAtIndex:i];
if((NSNull *)controller == [NSNull null])
{
    //
}

//判断字典对象的元素是否为空
NSString *userId=[NSDictionary objectForKey:@"UserId"];
if(userId == [NSNull null])
{
    //
}
```