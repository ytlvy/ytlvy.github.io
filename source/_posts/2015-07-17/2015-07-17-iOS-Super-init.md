title: iOS Super init
categories: IOS
tags:
  - init
date: 2015-07-17 16:25:41
author:
description:
photos:
---
[转自](http://www.cocoawithlove.com/2009/04/what-does-it-mean-when-you-assign-super.html)

# What does it mean when you assign [super init] to self?
`self = [super init];` 是 Objctive-C 语法中很奇怪的一句.

## Converting a method invocation
首先需要了解`self`参数, 编译器是如何处理的.当你输入下面代码时

```
MyClass *myObject = [[MyClass alloc] initWithString:@"someString"];
```

编译器会转换为大致如下的代码:

```
class myClass = objc_getClass("MyClass");
SEL allocSelector = @selector(alloc);
MyClass *myObject1 = objc_msgSend(myClass, allocSelector);

SEL initSelector = @selector(initWithString:);
MyClass *myObject2 = objc_msgSend(myObject1, initSelector, @"someString");
```

## So what is "self"?
每个方法都有两个隐藏参数: `self` and `_cmd`.

例如:

```
- (id)initWithString:(NSString *)aString;
```

会转换为:

```
id initWithString(id self, SEL _cmd, NSString *aString);
```

实际上`self` 只是在每个方法中都存在的隐藏参数, 像普通参数一样, 它通过函数调用来接收数值.

你可以尝试将下面的方法变更为`objc_msgSend`方式来调用

```
[myObject someMethodWithParameter:someValue];
```

你可以直接调用

```
SEL methodSelector = @selector(someMethodWithParameter:);
IMP someMethodFunction = class_getMethodImplementation([myObject class], methodSelector);
someMethodFunction(myObject, methodSelector, someValue);
```

`self`正是由于接收了函数`someMethodFunction`传入的第一个参数, 才拥有了实际的数值, 如果
你此时传入其他的数值, 很可能会造成系统崩溃.

## Why have a "self" parameter at all?
方法需要知道它操作的数据是什么, 而` self ` 正是用来告诉类它要操作的数据是什么的. 因为在运行时, `self `也即是对象本身存贮了数据, 而方法使在类中存储的. 所以方法操作数据的时候, 需要通过 `self` 指针来获取数据的具体位置.例如:

```
@interface MyClass : NSObject
{
    NSInteger value;
}
- (void)setValueToZero;
@end
```

方法:

```
- (void)setValueToZero
{
    value = 0;
}
```

转换为:

```
void setValueToZero(id self, SEL _cmd)
{
    self->value = 0;
}
```

## So does self already have a value when init is called?

从上面的分析看, `initWithString` 是 方法调用`[[MyClass alloc] initWithString:@"someString"]`的一部分.

```
MyClass *myObject2 = objc_msgSend(myObject1, initSelector, @"someString");
```

所以当我们进入`initWithString`方法时, `self` 已经有值`myObject1`(通过 `[MyClass alloc]`  返回的), 因为没有` super `的调用是需要` self `的,
```
[super init];
```

会转换为:

```
struct objc_super super = {.receiver = self, .super_class = 0xC0FFEE};
objc_msgSendSuper(&super, @selector(otherMethodWithArgument:), arg);
```

## So why assign the value returned from [super init] to self?

```
- (id)initWithString:(NSString *)aString
{
    self = [super init];
    if (self)
    {
        instanceString = [aString retain];
    }
    return self;
}
```

为什么需要将 `self` 重新赋值为 `[super init]`, 原因为`[super init] ` 会发生下面3种情况之一:

1. Return its own receiver (the self pointer doesn't change) with inherited instance values initialized.
2. Return a different object with inherited instance values initialized.
3. Return nil, indicating failure.

第一种情况, 赋值是没有意义的, 因为 self 没有改变,  `instanceString` 是作用在原来的对象上的.
第三种情况, 初始化失败返回` nil `, ` self ` 也被赋值为` nil `并返回, 不会发生更进一步的操作.
第二种情况, 才是重新赋值的意义所在. 因为返回的对象改变了.


## It's almost never required to initialize self

所以重新赋值的意义在于`[super init]`可能会返回一个不同的对象.那什么时候 会返回不同对象呢, 情况如下:
1. Singleton object (always returns the singleton instead of any subsequent allocation)
2. Other unique objects ([NSNumber numberWithInteger:0] always returns the global "zero" object)
3. Class clusters substitute private subclasses when you initialize an instance of the superclass.
4. Classes which choose to reallocate the same (or compatible) class based on parameters passed into the initializer.

除了最后一种情况外, 如果继续初始化返回的对象, 其实是错误的, 因为这个对象已经被完全初始化完毕了.
所以上面说的`[super init] ` 可能发生的情况, 可以扩充了4种:

1. Return its own receiver (the self pointer doesn't change) with inherited instance values initialized.
2. Return an object of the same class, requiring further initialization.
3. Return a different object that is already completely initialized.
4. Return nil, indicating failure.

上面列表中, 第二种和第三种情况是冲突的, 我们原来典型的写法` self = [super init]` 可以解决1, 2, 4等情况.
而可以解决1, 3, 4情况的写法为:
```
- (id)initWithString:(NSString *)aString
{
    id result = [super init];
    if (self == result)
    {
        instanceString = [aString retain];
    }
    return result;
}
```

因为类簇, 单例和唯一对象等情况都符合第三种情况, 如此以来, 系统中大量的类放入了这种情况, 我仅知道`NSManagedObject`符合第二种情况.奇怪的是, 适用范围更广的上面的写法反而不是标准写法.

## 总结
大部分情况下, 你不需要将`[ super init]` 重新赋值给`self`, 在某些情况下, 这样做更是错误的.







