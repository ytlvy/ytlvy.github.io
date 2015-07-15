title: 'iOS Runtime Object & Class & Meta Class'
categories: IOS
tags:
  - IOS
  - Meta Class
description:
date: 2015-07-13 21:19:42
author:
photos:
---
### 习题内容
下面代码的运行结果是?
```
@interface Sark : NSObject
@end

@implementation Sark
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
        BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];

        BOOL res3 = [(id)[Sark class] isKindOfClass:[Sark class]];
        BOOL res4 = [(id)[Sark class] isMemberOfClass:[Sark class]];

        NSLog(@"%d %d %d %d", res1, res2, res3, res4);
    }
    return 0;
}
```

运行结果为：

```
2014-11-05 14:45:08.474 Test[9412:721945] 1 0 0 0
```
<!-- more -->
### 什么是 id
id 在 objc.h 中定义如下:
```
/// A pointer to an instance of a class.
typedef struct objc_object *id;
```

就像注释中所说的这样 id 是指向一个 objc_object 结构体的指针。
> id 这个struct的定义本身就带了一个 *, 所以我们在使用其他NSObject类型的实例时需要在前面加上 *， 而使用 id 时
> 却不用。

### 那么objc_object又是什么呢
objc_object 在 objc.h 中定义如下:
```
/// Represents an instance of a class.
struct objc_object {
    Class isa;
};
```

这个时候我们知道Objective-C中的object在最后会被转换成C的结构体，而在这个struct中有一个 isa 指针，指向它的类别 Class。

### 那么什么是Class呢
在 objc.h 中定义如下:
```
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;
```

我们可以看到 Class本身指向的也是一个C的`struct objc_class`。

继续看在runtime.h中`objc_class`定义如下:
```
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
    #if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
    #endif
} OBJC2_UNAVAILABLE;
```

该结构体中，isa 指向所属Class， super_class指向父类别。

下载objc源代码，在 objc-runtime-new.h 中，我们发现 objc_class有如下定义:
```
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;   
    ...
    ...
}
```

豁然开朗，我们看到在Objective-C的设计哲学中，一切都是对象。Class在设计中本身也是一个对象。而这个Class对象的对应的类，我们叫它 `Meta Class`。即Class结构体中的 isa 指向的就是它的 Meta Class。

### Meta Class
根据上面的描述，我们可以把Meta Class理解为 一个Class对象的Class。简单的说：

- 当我们发送一个消息给一个NSObject对象时，这条消息会在对象的类的方法列表里查找
- 当我们发送一个消息给一个类时，这条消息会在类的Meta Class的方法列表里查找

而 Meta Class本身也是一个Class，它跟其他Class一样也有自己的 isa 和 super_class 指针。看下图：
![](http://106.186.113.24:8888/other/Class%26MetaClass.001.jpg)

- 每个Class都有一个isa指针指向一个唯一的Meta Class
- 每一个Meta Class的isa指针都指向最上层的`Meta Class`（图中的NSObject的Meta Class）
- 最上层的`Meta Class`的`isa`指针指向自己，形成一个回路
- 每一个`Meta Class`的`super class`指针指向它原本Class的 `Super Class`的`Meta Class`。但是最上层的`Meta Class`的 `Super Class`指向`NSObject Class`本身
- 最上层的NSObject Class的super class指向 nil

### 解惑
为了更加清楚的知道整个函数调用过程，我们使用`clang -rewrite-objc main.m`重写，可获得如下代码：
```
BOOL res1 = ((BOOL (*)(id, SEL, Class))(void *)objc_msgSend)((id)((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("class")), sel_registerName("isKindOfClass:"), ((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("class")));

 BOOL res2 = ((BOOL (*)(id, SEL, Class))(void *)objc_msgSend)((id)((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("class")), sel_registerName("isMemberOfClass:"), ((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("class")));

 BOOL res3 = ((BOOL (*)(id, SEL, Class))(void *)objc_msgSend)((id)((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("Sark"), sel_registerName("class")), sel_registerName("isMemberOfClass:"), ((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("class")));

 BOOL res4 = ((BOOL (*)(id, SEL, Class))(void *)objc_msgSend)((id)((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("Sark"), sel_registerName("class")), sel_registerName("isMemberOfClass:"), ((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("class")));

```

先看前两个调用：
- 最外层是 objc_msgSend函数，转发消息。
- 函数第一个参数是 `(id)((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("class"))`
- 函数第二个参数是转发的selector
- 函数第三个参数是 `((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("class"))`

我们注意到第一个参数和第三个参数对应重写的是[NSObject class]，即使用`objc_msgSend`向 `NSObject Class` 发送 `@selector(class)` 这个消息

打开objc源代码，在 Object.mm 中发现`+ (Class)class`实现如下:
```
+ (Class)class {
    return self;
}
```

所以即返回Class类的对象本身。看如下输出:
```
NSLog(@"%p", [NSObject class]);
NSLog(@"%p", [NSObject class]);

2014-11-05 18:48:30.939 Test[11682:865988] 0x7fff768d40f0
2014-11-05 18:48:30.940 Test[11682:865988] 0x7fff768d40f0
```

继续打开objc源代码，在 `Object.mm` 中，我们发现 `isKindOfClass`的实现如下:
```
- (BOOL)isKindOf:aClass
{
    Class cls;
    for (cls = isa; cls; cls = cls->superclass) 
        if (cls == (Class)aClass)
            return YES;
    return NO;
}
```

对着上面Meta Class的图和实现，我们可以看出

- 当 NSObject Class对象第一次进行比较时，得到它的isa为 NSObject的Meta Class， 这个时候 NSObject Meta Class 和 NSObject Class不相等。
- 然后取NSObject 的Meta Class 的Super class，这个时候又变成了 NSObject Class， 所以返回相等

所以上述第一个输出结果是 YES 。

我们在看下 ‘isMemberOfClass’的实现:
```
- (BOOL)isMemberOf:aClass
{
    return isa == (Class)aClass;
}
```

综上所述，当前的 isa 指向 NSObject 的 `Meta Class`， 所以和 `NSObject Class`不相等。

所以上述第二个输出结果为 NO 。

继续看后面两个调用:

- Sark Class 的isa指向的是 Sark的Meta Class，和Sark Class不相等
- Sark Meta Class的super class 指向的是 NSObject Meta Class， 和 Sark Class不相等
- NSObject Meta Class的 super class 指向 NSObject Class，和 Sark Class 不相等
- NSObject Class 的super class 指向 nil， 和 Sark Class不相等

所以后面两个调用的结果都输出为 NO 。














