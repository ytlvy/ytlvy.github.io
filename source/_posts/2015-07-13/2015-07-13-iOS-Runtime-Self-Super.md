title: 'iOS Runtime Self & Super'
categories: IOS
tags:
  - IOS
  - Self
  - Super
description:
date: 2015-07-13 21:18:00
author:
photos:
---
## Self && Super

### What is Super?
`Super` 是和 `Self` 一样的关键字, 不同的是, 它不能用作函数的参数, 只能接收消息.当接收消息的时候, 它会向父类搜索方法的定义. 如果在向上的类链中没有任何定义, 则程序会 crash.

```
- (instancetype)initWithValue1:(id)value1 value2:(id)value2
{
    self = [super initWithValue1:value1];

    if (self) {
        _value2 = value2;
    }

    return self;
}
```

### messaging `super`
```
struct objc_super {
    __unsafe_unretained id receiver;
    __unsafe_unretained Class super_class;
};
```

```
- (void)methodWithArgument:(id)arg
{
    [super otherMethodWithArgument:arg];
}
```

```
- (void)methodWithArgument:(id)arg
{
    struct objc_super super = {.receiver = self, .super_class = 0xC0FFEE};

    objc_msgSendSuper(&super, @selector(otherMethodWithArgument:), arg);
}
```

`objc_msgSendSuper` 将从父类的定义开始查找方法的实现. `super_class`是在 runtime 时, 动态生成的.
<!-- more -->
#### objc_msgSendSuper 例子

![](http://cdn.macoscope.com/blog/wp-content/uploads/2015/02/how-does-super-work-1.png)

```
#import <Foundation/Foundation.h>
#import <objc/message.h>

@interface A : NSObject @end
@interface B : A @end
@interface C : B @end
@interface X : NSObject @end

@implementation A 
- (void)abc
{
    NSLog(@"-[A abc] called from class %@", NSStringFromClass([self class]));
}
@end

@implementation B
@end

@implementation C
- (void)abc
{
    NSLog(@"-[C abc] called from class %@", NSStringFromClass([self class]));
}
@end

@implementation X
-(void)A_abc
{
    struct objc_super sup = {self, [A class]};
    objc_msgSendSuper(&sup, @selector(abc));
}
-(void)B_abc
{
    struct objc_super sup = {self, [B class]};
    objc_msgSendSuper(&sup, @selector(abc));
}
-(void)C_abc
{
    struct objc_super sup = {self, [C class]};
    objc_msgSendSuper(&sup, @selector(abc));
}
@end

int main(int argc, char *argv[])
{
    @autoreleasepool {
        [[X new] A_abc];
        [[X new] B_abc];
        [[X new] C_abc];
    }
}
```

Output:

```
-[A abc] called from class X
-[A abc] called from class X
-[C abc] called from class X
```

#### Message Forwarding Experiment

![](http://cdn.macoscope.com/blog/wp-content/uploads/2015/02/how-does-super-work-2.png)
```
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

@interface NSInvocation (PrivateAPI)
- (void)invokeUsingIMP:(IMP)imp;
@end

@interface A : NSObject @end
@interface B : A @end
@interface C : B @end
@interface X : NSObject @end

@implementation A 
- (void)abc
{
    NSLog(@"-[A abc]");
}
@end

@implementation B
@end

@implementation C
- (void)abc
{
    [super abc];
    NSLog(@"-[C abc]");
}
@end

@interface X (ForwardedMethods)
- (void)xyz;
@end

@implementation X
- (BOOL)respondsToSelector:(SEL)selector
{
    return selector == @selector(xyz);
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector
{
    if (selector == @selector(xyz)) {
        return [C instanceMethodSignatureForSelector:@selector(abc)];
    }

    return nil;
}

- (void)forwardInvocation:(NSInvocation *)invocation
{
    if (invocation.selector == @selector(xyz)) {
        Method method = class_getInstanceMethod([C class], @selector(abc));
        [invocation invokeUsingIMP:method_getImplementation(method)];
    }
}
@end

int main(int argc, char *argv[])
{
    @autoreleasepool {
        [[X new] xyz];  
    }
}
```

Output:
```
-[A abc]
-[C abc]
```

#### Method Copying Experiment

![](http://cdn.macoscope.com/blog/wp-content/uploads/2015/02/how-does-super-work-3.png)
```
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

@interface A : NSObject @end
@interface B : A @end
@interface C : B @end
@interface X : NSObject @end

@implementation A 
- (void)abc
{
    NSLog(@"-[A abc]");
}
@end

@implementation B
@end

@implementation C
- (void)abc
{
    [super abc];
    NSLog(@"-[C abc]");
}
@end

@interface X (CopiedMethods)
- (void)xyz;
@end

@implementation X
+ (void)load
{
    Method method = class_getInstanceMethod([C class], @selector(abc));
    IMP imp = method_getImplementation(method);
    const char *typeEncoding = method_getTypeEncoding(method);
    class_addMethod(self, @selector(xyz), imp, typeEncoding);
}
@end

int main(int argc, char *argv[])
{
    @autoreleasepool {
        [[X new] xyz];  
    }
}
```

Output:
```
-[A abc]
-[C abc]
```


### 刨根问底
```
@implementation Son : Father
- (id)init
{
    self = [super init];
    if (self)
    {
        NSLog(@"%@", NSStringFromClass([self class]));
        NSLog(@"%@", NSStringFromClass([super class]));
    }
    return self;
}
@end
```

解惑：这个题目主要是考察关于objc中对 self 和 super 的理解。

self 是类的隐藏参数，指向当前调用方法的这个类的实例。而 super 是一个 Magic Keyword， 它本质是一个编译器标示符，和 self 是指向的同一个消息接受者。上面的例子不管调用[self class]还是[super class]，接受消息的对象都是当前 Son ＊xxx 这个对象。而不同的是，super是告诉编译器，调用 class 这个方法时，要去父类的方法，而不是本类里的。

当使用 self 调用方法时，会从当前类的方法列表中开始找，如果没有，就从父类中再找；而当使用 super 时，则从父类的方法列表中开始找。然后调用父类的这个方法。

真的是这样吗？继续看：

使用clang重写命令:
```
$ clang -rewrite-objc test.m
```

发现上述代码被转化为:

```
NSLog( (NSString *) &__NSConstantStringImpl__var_folders_gm_0jk35cwn1d3326x0061qym280000gn_T_main_a5cecc_mi_0, 
    NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("class"))) );

NSLog( (NSString *) &__NSConstantStringImpl__var_folders_gm_0jk35cwn1d3326x0061qym280000gn_T_main_a5cecc_mi_1, 
    NSStringFromClass(((Class (*)(__rw_objc_super *, SEL))(void *)objc_msgSendSuper)((__rw_objc_super){ (id)self, (id)class_getSuperclass(objc_getClass("Son")) }, sel_registerName("class"))) );
```

精简后
```
objc_msgSend(self,  @selector"class")

objc_msgSendSuper( {self, class_getSuperclass(objc_getClass("Son")),  @selector"class"}
```

从上面的代码中，我们可以发现在调用 `[self class] `时，会转化成 `objc_msgSend`函数。看下函数定义：

```
id objc_msgSend(id self, SEL op, ...)
```

我们把 `self` 做为第一个参数传递进去。

而在调用 `[super class]`时，会转化成 `objc_msgSendSuper`函数。看下函数定义:

```
id objc_msgSendSuper(struct objc_super *super, SEL op, ...)
```

第一个参数是 `objc_super` 这样一个结构体，其定义如下:

```
struct objc_super {
   __unsafe_unretained id receiver;
   __unsafe_unretained Class super_class;
};
```

结构体有两个成员，第一个成员是 `receiver`, 类似于上面的 `objc_msgSend`函数第一个参数`self` 。第二个成员是记录当前类的父类是什么。

所以，当调用 `[self class] `时，实际先调用的是 `objc_msgSend`函数，第一个参数是 `Son`当前的这个实例，然后在 `Son` 这个类里面去找 `- (Class)class`这个方法，没有，去父类 Father里找，也没有，最后在 `NSObject` 类中发现这个方法。而 `- (Class)class`的实现就是返回self的类别，故上述输出结果为 `Son`。

objc Runtime开源代码对- (Class)class方法的实现:
```
- (Class)class {
    return object_getClass(self);
}
```

而当调用 `[super class]`时，会转换成`objc_msgSendSuper`函数。第一步先构造` objc_super` 结构体，结构体第一个成员就是 `self` 。第二个成员是 `(id)class_getSuperclass(objc_getClass(“Son”)) `, 实际该函数输出结果为 Father。第二步是去  `Father` 这个类里去找 `- (Class)class` ，没有，然后去NSObject类去找，找到了。最后内部是使用 objc_msgSend(objc_super->receiver, @selector(class))去调用，此时已经和[self class]调用相同了，故上述输出结果仍然返回 Son


### 总结
`super`的作用主要是告诉编译器 方法的定义从父类开始查找, 并不是生成了一个真的父类对象. 而 ` class` 方法的定义`return object_getClass(self);` 决定了其返回结果是运行期动态决定的, 而此时的` self ` 变量同样只能是` Son `的实例对象.
