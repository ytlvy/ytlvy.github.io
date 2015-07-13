title: iOS Runtime 成员变量与属性
categories: IOS
tags:
  - IOS
  - Property
  - iVar
description:
date: 2015-07-13 21:20:33
author:
photos:
---

### 习题内容
下面代码会? Compile Error / Runtime Crash / NSLog…?
```
@interface Sark : NSObject
@property (nonatomic, copy) NSString *name;
@end

@implementation Sark

- (void)speak {
    NSLog(@"my name is %@", self.name);
}

@end

@interface Test : NSObject
@end

@implementation Test

- (instancetype)init {
    self = [super init];
    if (self) {
        id cls = [Sark class];
        void *obj = &cls;
        [(__bridge id)obj speak];
    }
    return self;
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        [[Test alloc] init];
    }
    return 0;
}
```

答案：代码正常输出，输出结果为：

```
2014-11-07 14:08:25.698 Test[1097:57255] my name is <Test: 0x1001002d0>
```
<!-- more -->
### 为什么呢?
前几节博文中多次讲到了`objc_class`结构体，今天我们再拿出来看一下：

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

其中`objc_ivar_list`结构体存储着`objc_ivar`数组列表，而`objc_ivar`结构体存储了类的单个成员变量的信息。

那么什么是Ivar呢？

Ivar 在objc中被定义为：

```
typedef struct objc_ivar *Ivar;
```

它是一个指向objc_ivar结构体的指针，结构体有如下定义：
```
struct objc_ivar {
    char *ivar_name                                          OBJC2_UNAVAILABLE;
    char *ivar_type                                          OBJC2_UNAVAILABLE;
    int ivar_offset                                          OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
}                                                            OBJC2_UNAVAILABLE;
```

这里我们注意第三个成员 `ivar_offset`。它表示基地址偏移字节。

在编译我们的类时，编译器生成了一个` ivar`布局，显示了在类中从哪可以访问我们的 ivars 。看下图:

![](http://106.186.113.24:8888/other/2014031602.png)

上图中，左侧的数据就是地址偏移字节，我们对 ivar 的访问就可以通过 `对象地址 ＋ ivar偏移字节`的方法。但是这又引发一个问题，看下图:

![](http://106.186.113.24:8888/other/2014031603.png)

我们增加了父类的ivar，这个时候布局就出错了，我们就不得不重新编译子类来恢复兼容性。

而Objective－C Runtime中使用了`Non Fragile ivars`，看下图:

![](http://106.186.113.24:8888/other/2014031604.png)

使用`Non Fragile ivars`时，Runtime会进行检测来调整类中新增的ivar的偏移量。 这样我们就可以通过 `对象地址 + ivar偏移字节`的方法来计算出ivar相应的地址，并访问到相应的ivar。

```
@interface Student : NSObject
{
    @private
    NSInteger age;
}
@end

@implementation Student
- (NSString *)description
{
    return [NSString stringWithFormat:@"age = %d", age];
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Student *student = [[Student alloc] init];
        student->age = 24;
    }
    return 0;
}
```

上述代码，Student有两个被标记为private的ivar，这个时候当我们使用 -> 访问时，编译器会报错。那么我们如何设置一个被标记为private的ivar的值呢?

通过上面的描述，我们知道ivar是通过计算字节偏量来确定地址，并访问的。我们可以改成这样:

```
@interface Student : NSObject  {
    @private
    int age;
}
@end

@implementation Student

- (NSString *)description {
    NSLog(@"current pointer = %p", self);
    NSLog(@"age pointer = %p", &age);
    return [NSString stringWithFormat:@"age = %d", age];
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Student *student = [[Student alloc] init];
        Ivar age_ivar = class_getInstanceVariable(object_getClass(student), "age");
        int *age_pointer = (int *)((__bridge void *)(student) + ivar_getOffset(age_ivar));
        NSLog(@"age ivar offset = %td", ivar_getOffset(age_ivar));
        *age_pointer = 10;
        NSLog(@"%@", student);
    }
    return 0;
}
```

上述代码的输出结果为:

```
2014-11-08 18:24:38.892 Test[4143:466864] age ivar offset = 8
2014-11-08 18:24:38.893 Test[4143:466864] current pointer = 0x1001002d0
2014-11-08 18:24:38.893 Test[4143:466864] age pointer = 0x1001002d8
2014-11-08 18:24:38.894 Test[4143:466864] age = 10
```

我们可以清晰的看到指针地址的变化和偏移量，和我们上述描述一致

### 说完了Ivar， 那Property又是怎么样的呢？
使用 `clang -rewrite-objc main.m` 重写题目中的代码，我们发现`Sark`类中的`name`属性被转换成了如下代码:
```
struct Sark_IMPL {
    struct NSObject_IMPL NSObject_IVARS;
    NSString *_name;
};

// @property (nonatomic, copy) NSString *name;
/* @end */


// @implementation Sark

static NSString * _I_Sark_name(Sark * self, SEL _cmd) { return (*(NSString **)((char *)self + OBJC_IVAR_$_Sark$_name)); }

static void _I_Sark_setName_(Sark * self, SEL _cmd, NSString *name) { objc_setProperty (self, _cmd, __OFFSETOFIVAR__(struct Sark, _name), (id)name, 0, 1); }
```

类中的 `Property` 属性被编译器转换成了`Ivar`，并且自动添加了我们熟悉的`Set`和`Get`方法。

我们这个时候回头看一下`objc_class`结构体中的内容，并没有发现用来专门记录`Property`的list。我们翻开objc源代码，在`objc-runtime-new.h`中，发现最终还是会通过在`class_ro_t`结构体中使用property_list_t存储对应的propertyies。

而在刚刚重写的代码中，我们可以找到这个`property_list_t`:
```
static struct /*_prop_list_t*/ {
    unsigned int entsize;  // sizeof(struct _prop_t)
    unsigned int count_of_properties;
    struct _prop_t prop_list[1];
    } _OBJC_$_PROP_LIST_Sark __attribute__ ((used, section ("__DATA,__objc_const"))) = {
        sizeof(_prop_t),
        1,
        name
};

static struct _class_ro_t _OBJC_CLASS_RO_$_Sark __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    0, __OFFSETOFIVAR__(struct Sark, _name), sizeof(struct Sark_IMPL), 
    (unsigned int)0, 
    0, 
    "Sark",
    (const struct _method_list_t *)&_OBJC_$_INSTANCE_METHODS_Sark,
    0, 
    (const struct _ivar_list_t *)&_OBJC_$_INSTANCE_VARIABLES_Sark,
    0, 
    (const struct _prop_list_t *)&_OBJC_$_PROP_LIST_Sark,
};
```

### 解惑
1）为什么能够正常运行，并调用到`speak`方法？
```
id cls = [Sark class];
void *obj = &cls;
[(__bridge id)obj speak];
```

obj被转换成了一个指向`cls`的指针，而 `cls` 又指向`[Sark Class]`, 此时` cls` 起到了类似` isa` 指针的作用, 因为` isa`也是指向一个类. 之后当`obj`被强制转换为`objc_object`类型时, `obj`就被当做了一个`[Sark Class]`的对象, 但是此对象和实际的对象是有差别的, 实际的对象时分配在堆上的, 而这个变量是临时变量, 本身分配在栈上, 强制转换后, 也同样还在栈上. 同时此对象除了有` isa ` 指针相对应的类能提供的特性外, 其他的一切都是假的.

也正因为` obj` 此时被强制转换为`[Sark Class]`, 所以它可以调用` speak`方法.


2) 为什么self.name的输出为 <Test: 0x1001002d0> ?
在` speak`方法中, 关于` self.name`的调用, 由上面的` ivar` 的获取方法`对象地址 + ivar偏移字节`,  可以推导出此时 self.name的调用地址为 `obj`指向对象的地址 + 偏移8(64位系统)而得出.  

`obj`指向的对象地址, 由上面的代码`void *obj = &cls;` 可知, 是`cls`变量的地址, 而此局部变量是分配在栈上的. 同时所有的局部变量是按照出现顺序从高到底一个个入栈的.可以参考[这篇文章](/2015/07/06/progress-memory-map/). 而 `cls` 变量地址+8是 `cls`局部变量上一个局部变量.

具体的情况,我们在下面来分析

我们在测试代码中加入一些调试代码和Log如下:
```
- (void)speak
{ 
    unsigned int numberOfIvars = 0;
    Ivar *ivars = class_copyIvarList([self class], &numberOfIvars);
    for(const Ivar *p = ivars; p < ivars+numberOfIvars; p++) {
        Ivar const ivar = *p;
        ptrdiff_t offset = ivar_getOffset(ivar);
        const char *name = ivar_getName(ivar);
        NSLog(@"Sark ivar name = %s, offset = %td", name, offset);
    }
    NSLog(@"_name变量 本身的地址 %p", &_name);
    NSLog(@"_name变量 指向地址 %p", _name);
    NSLog(@"_name 变量指向对象的内容 %@", _name);
}

@implementation Test

- (instancetype)init
{
   self = [super init];

    if (self) {
        
        NSLog(@"=================================");
        NSLog(@"self变量 指向对象的地址 = %p ========", self);
        NSLog(@"self变量 本身的地址 = %p", &self);
        NSLog(@"=================================");
        
        NSLog(@"select address %p", @selector(initWithAttribute:));
        NSLog(@"_cmd address %p", _cmd);

        id cls = [Sark class];
        NSLog(@"cls变量 指向的地址 = %p", cls);
        NSLog(@"cls变量 本身地址 = %p", &cls);
        NSLog(@"=================================");
        
        void *obj = &cls;
        NSLog(@"obj变量 指向的虚假对象地址 = %@", obj);
        NSLog(@"obj变量 本身地址 = %p", &obj);
        NSLog(@"=================================");

        [(__bridge id)obj speak];
        
    }
    return self;
}

@end
```

输出结果如下:

```
2015-07-13 20:24:31.485 iTest[13990:495035] =================================
2015-07-13 20:24:31.487 iTest[13990:495035] self变量 指向对象的地址 = 0x7f8d32700050 ========
2015-07-13 20:24:31.487 iTest[13990:495035] self变量 本身的地址 = 0x7fff5cb8f408
2015-07-13 20:24:31.487 iTest[13990:495035] =================================
2015-07-13 20:24:31.488 iTest[13990:495035] select address 0x103071b4e
2015-07-13 20:24:31.488 iTest[13990:495035] _cmd address 0x10603733b
2015-07-13 20:24:31.489 iTest[13990:495035] cls变量 指向的地址 = 0x103075760
2015-07-13 20:24:31.489 iTest[13990:495035] cls变量 本身地址 = 0x7fff5cb8f3e8
2015-07-13 20:24:31.489 iTest[13990:495035] =================================
2015-07-13 20:24:31.490 iTest[13990:495035] obj变量 指向的虚假对象地址 = <Sark: 0x7fff5cb8f3e8>
2015-07-13 20:24:31.490 iTest[13990:495035] obj变量 本身地址 = 0x7fff5cb8f3e0
2015-07-13 20:24:31.490 iTest[13990:495035] =================================
2015-07-13 20:24:31.491 iTest[13990:495035] Sark ivar name = _name, offset = 8
2015-07-13 20:24:31.491 iTest[13990:495035] _name变量 本身的地址 0x7fff5cb8f3f0
2015-07-13 20:24:31.492 iTest[13990:495035] _name变量 指向地址 0x7f8d32700050
2015-07-13 20:24:31.492 iTest[13990:495035] _name 变量指向对象的内容 <Test: 0x7f8d32700050>
```

由上面的分析可知 `_name`指针本身的地址为`0x7fff5cb8f3f0` 正好是指针`cls`本身地址`0x7fff5cb8f3e8` + `8`得到的。而此地址由Debug 输出可知为`<Test: 0x7f8d32700050>`. 但是我们发现`cls`的上一个变量是` self`, 而` self` 的地址确是`0x7fff5cb8f408`, 是` cls`地址+`32`.

![](http://7jpswx.com1.z0.glb.clouddn.com/iOS%20Runtime%20成员变量与属性1.png)
![](http://7jpswx.com1.z0.glb.clouddn.com/iOS%20Runtime%20成员变量与属性2.png)

通过在Xcode自带的内存浏览器中查看, 并通过 `LLDB` 打印输出3个多出的变量, 分别为 `Test对象指针地址`, `Test类`, `_cmd`. 

### 进一步分析
将代码注释如下:
```
//    self = [super init];
//
//    if (self) {

    ....

    //    }
//    return self;
    return nil;
```

在`NSLog(@"cls变量 本身地址 = %p", &cls);`句下面断点, 输出如下:
```
2015-07-13 20:43:05.377 iTest[14251:506168] =================================
2015-07-13 20:43:05.378 iTest[14251:506168] self变量 指向对象的地址 = 0x7fda40f02c20 ========
2015-07-13 20:43:05.378 iTest[14251:506168] self变量 本身的地址 = 0x7fff5f003408
2015-07-13 20:43:05.378 iTest[14251:506168] =================================
2015-07-13 20:43:05.379 iTest[14251:506168] select address 0x100bfdb4e
2015-07-13 20:43:05.379 iTest[14251:506168] _cmd address 0x103bc333b
2015-07-13 20:43:05.379 iTest[14251:506168] cls变量 指向的地址 = 0x100c01758
2015-07-13 20:43:05.379 iTest[14251:506168] cls变量 本身地址 = 0x7fff5f0033f8
```

此时` cls ` 和` self`相差 `0x7fff5f003408 - 0x7fff5f0033f8 = 16` 恰好为两个变量地址, 而我们由 runtime 调用知道, 每个方法都默认隐藏传入了两个变量` self` 和 `_cmd`. 从内存浏览器可以看出此时 `self`变量的上一个地址 `0x7fff5f003400`恰好为`0x103bc333b` 是`_ cmd`的地址. 而此时由于` self.name `指向了 `_cmd`的地址, 程序继续运行报错`EXC_BAD_ACCESS`
![](http://7jpswx.com1.z0.glb.clouddn.com/iOS%20Runtime%20成员变量与属性3.png)

由此可以推论出:
1) `_cmd` 为隐藏参数自动传入, 

2)`Test类`和另外一个`Test对象指针地址` 应该是 `[super init]`语句产生的, 经过编译后, `objc_super`结构体正好包含两个变量分别为`(id)self`, `objc_getClass("Test")`.

```
self = ((Test *(*)(__rw_objc_super *, SEL, NSInteger))(void *)objc_msgSendSuper)((__rw_objc_super){ (id)self, (id)class_getSuperclass(objc_getClass("Test")) }, sel_registerName("init"));
```

看下图，可以清楚的展示整个计算的过程：

![](http://7jpswx.com1.z0.glb.clouddn.com/iOS%20Runtime%20成员变量与属性.jpg)

我们可以做一个另外的实验，把`Test Class` 的`init`方法改为如下代码：

```
@interface Father : NSObject
@end

@implementation Father
@end

@implementation Test

- (instancetype)init
{
    self = [super init];
    if (self) {
        NSLog(@"self变量 指向对象的地址 = %p ========", self);
        NSLog(@"self变量 本身的地址 = %p", &self);
        NSLog(@"=================================");
        
        NSString *ss = @"hello world";
        NSLog(@"ss变量 指向对象的地址 = %p ========", ss);
        NSLog(@"ss变量 本身的地址 = %p", &ss);
        NSLog(@"=================================");

        id cls = [Sark class];
        NSLog(@"cls变量 指向的地址 = %p", cls);
        NSLog(@"cls变量 本身地址 = %p", &cls);
        NSLog(@"=================================");
        
        void *obj = &cls;
        NSLog(@"obj变量 指向的虚假对象地址 = %@", obj);
        NSLog(@"obj变量 本身地址 = %p", &obj);
        NSLog(@"=================================");

        [(__bridge id)obj speak];
    }
    return self;
}

@end
```

你会发现这个时候的输出变成了:

```
2015-07-13 14:16:23.281 iTest[11586:381830] self变量 指向对象的地址 = 0x7fa3325007b0 ========
2015-07-13 14:16:23.283 iTest[11586:381830] self变量 本身的地址 = 0x7fff57c6e408
2015-07-13 14:16:23.283 iTest[11586:381830] =================================
2015-07-13 14:16:23.284 iTest[11586:381830] ss变量 指向对象的地址 = 0x107f944e0 ========
2015-07-13 14:16:23.284 iTest[11586:381830] ss变量 本身的地址 = 0x7fff57c6e3e8
2015-07-13 14:16:23.284 iTest[11586:381830] =================================
2015-07-13 14:16:23.285 iTest[11586:381830] cls变量 指向的地址 = 0x107f966d8
2015-07-13 14:16:23.285 iTest[11586:381830] cls变量 本身地址 = 0x7fff57c6e3e0
2015-07-13 14:16:23.286 iTest[11586:381830] =================================
2015-07-13 14:16:23.286 iTest[11586:381830] obj变量 指向的虚假对象地址 = <Sark: 0x7fff57c6e3e0>
2015-07-13 14:16:23.286 iTest[11586:381830] obj变量 本身地址 = 0x7fff57c6e3d8
2015-07-13 14:16:23.287 iTest[11586:381830] =================================
2015-07-13 14:16:23.287 iTest[11586:381830] Sark ivar name = _name, offset = 8
2015-07-13 14:16:23.287 iTest[11586:381830] _name变量 本身的地址 0x7fff57c6e3e8
2015-07-13 14:16:23.288 iTest[11586:381830] _name变量 指向地址 0x107f944e0
2015-07-13 14:16:23.288 iTest[11586:381830] _name 变量指向对象的内容 hello world
```


### 参考

http://chun.tips/blog/2014/11/08/bao-gen-wen-di-objective%5Bnil%5Dc-runtime(4)%5Bnil%5D-cheng-yuan-bian-liang-yu-shu-xing/
