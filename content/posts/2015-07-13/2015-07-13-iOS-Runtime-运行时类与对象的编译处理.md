---
author: null
categories:
- IOS
date: '2015-07-13'
description: null
photos: null
tags:
- IOS
- runtime
title: iOS Runtime 运行时类与对象的编译处理
---

## iOS Runtime运行时类与对象的编译处理

Objective-C语言是一门动态语言，它将很多静态语言在编译和链接时期做的事放到了运行时来处理。这种动态语言的优势在于：我们写代码时更具灵活性，如我们可以把消息转发给我们想要的对象，或者随意交换一个方法的实现等。
这种特性意味着`Objective-C`不仅需要一个编译器，还需要一个运行时系统来执行编译的代码。对于Objective-C来说，这个运行时系统就像一个操作系统一样：它让所有的工作可以正常的运行。这个运行时系统即Objc Runtime。Objc Runtime其实是一个Runtime库，它基本上是用C和汇编写的，这个库使得C语言有了面向对象的能力。

Runtime库主要做下面几件事：
1. 封装：在这个库中，对象可以用C语言中的结构体表示，而方法可以用C函数来实现，另外再加上了一些额外的特性。这些结构体和函数被runtime函数封装后，我们就可以在程序运行时创建，检查，修改类、对象和它们的方法了。
2. 找出方法的最终执行代码：当程序执行[object doSomething]时，会向消息接收者(object)发送一条消息(doSomething)，runtime会根据消息接收者是否能响应该消息而做出不同的反应。这将在后面详细介绍。

&nbsp;&nbsp;&nbsp;&nbsp;Objective-C runtime目前有两个版本：Modern runtime和Legacy runtime。Modern Runtime 覆盖了64位的Mac OS X Apps，还有 iOS Apps，Legacy Runtime 是早期用来给32位 Mac OS X Apps 用的，也就是可以不用管就是了。
在这一系列文章中，我们将介绍runtime的基本工作原理，以及如何利用它让我们的程序变得更加灵活。在本文中，我们先来介绍一下类与对象，这是面向对象的基础，我们看看在Runtime中，类是如何实现的。
<!--more-->
### 类与对象基础数据结构
#### Class
Objective-C类是由Class类型来表示的，它实际上是一个指向objc_class结构体的指针。它的定义如下：
```
typedef struct objc_class *Class;
```

查看objc/runtime.h中objc_class结构体的定义如下：
```
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                       OBJC2_UNAVAILABLE;  // 父类
    const char *name                        OBJC2_UNAVAILABLE;  // 类名
    long version                            OBJC2_UNAVAILABLE;  // 类的版本信息，默认为0
    long info                               OBJC2_UNAVAILABLE;  // 类信息，供运行期使用的一些位标识
    long instance_size                      OBJC2_UNAVAILABLE;  // 该类的实例变量大小
    struct objc_ivar_list *ivars            OBJC2_UNAVAILABLE;  // 该类的成员变量链表
    struct objc_method_list **methodLists   OBJC2_UNAVAILABLE;  // 方法定义的链表
    struct objc_cache *cache                OBJC2_UNAVAILABLE;  // 方法缓存
    struct objc_protocol_list *protocols    OBJC2_UNAVAILABLE;  // 协议链表
#endif

} OBJC2_UNAVAILABLE;
```

在这个定义中，下面几个字段是我们感兴趣的
1. isa：需要注意的是在Objective-C中，所有的类自身也是一个对象，这个对象的Class里面也有一个isa指针，它指向metaClass(元类)，我们会在后面介绍它。
2. super_class：指向该类的父类，如果该类已经是最顶层的根类(如NSObject或NSProxy)，则super_class为NULL。
3. cache：用于缓存最近使用的方法。一个接收者对象接收到一个消息时，它会根据isa指针去查找能够响应这个消息的对象。在实际使用中，这个对象只有一部分方法是常用的，很多方法其实很少用或者根本用不上。这种情况下，如果每次消息来时，我们都是methodLists中遍历一遍，性能势必很差。这时，cache就派上用场了。在我们每次调用过一个方法后，这个方法就会被缓存到cache列表中，下次调用的时候runtime就会优先去cache中查找，如果cache没有，才去methodLists中查找方法。这样，对于那些经常用到的方法的调用，但提高了调用的效率。
4. version：我们可以使用这个字段来提供类的版本信息。这对于对象的序列化非常有用，它可是让我们识别出不同类定义版本中实例变量布局的改变

针对cache，我们用下面例子来说明其执行过程：
```
NSArray *array = [[NSArray alloc] init];
```

其流程是：
1. [NSArray alloc]先被执行。因为NSArray没有+alloc方法，于是去父类NSObject去查找。
2. 检测NSObject是否响应+alloc方法，发现响应，于是检测NSArray类，并根据其所需的内存空间大小开始分配内存空间，然后把isa指针指向NSArray类。同时，+alloc也被加进cache列表里面。
3. 接着，执行-init方法，如果NSArray响应该方法，则直接将其加入cache；如果不响应，则去父类查找。
4. 在后期的操作中，如果再以[[NSArray alloc] init]这种方式来创建数组，则会直接从cache中取出相应的方法，直接调用。

> 备注上面的说法是错误的, NSArray 采用了类簇来实现, alloc 之后生成的`__NSPlacehodlerArray`不是NSArray


#### objc_object与id
objc_object是表示一个类的实例的结构体，它的定义如下(objc/objc.h)：
```
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};

typedef struct objc_object *id;
```


可以看到，这个结构体只有一个字体，即指向其类的isa指针。这样，当我们向一个Objective-C对象发送消息时，运行时库会根据实例对象的isa指针找到这个实例对象所属的类。Runtime库会在类的方法列表及父类的方法列表中去寻找与消息对应的selector指向的方法。找到后即运行这个方法。
当创建一个特定类的实例对象时，分配的内存包含一个objc_object数据结构，然后是类的实例变量的数据。NSObject类的alloc和allocWithZone:方法使用函数class_createInstance来创建objc_object数据结构。
另外还有我们常见的id，它是一个objc_object结构类型的指针。它的存在可以让我们实现类似于C++中泛型的一些操作。该类型的对象可以转换为任何一种对象，有点类似于C语言中void *指针类型的作用

#### objc_cache
上面提到了objc_class结构体中的cache字段，它用于缓存调用过的方法。这个字段是一个指向objc_cache结构体的指针，其定义如下
```
struct objc_cache {
    unsigned int mask /* total = mask + 1 */                 OBJC2_UNAVAILABLE;
    unsigned int occupied                                    OBJC2_UNAVAILABLE;
    Method buckets[1]                                        OBJC2_UNAVAILABLE;
};
```

该结构体的字段描述如下：
1. mask：一个整数，指定分配的缓存bucket的总数。在方法查找过程中，Objective-C runtime使用这个字段来确定开始线性查找数组的索引位置。指向方法selector的指针与该字段做一个AND位操作(index = (mask & selector))。这可以作为一个简单的hash散列算法。
2. occupied：一个整数，指定实际占用的缓存bucket的总数。
3. buckets：指向Method数据结构指针的数组。这个数组可能包含不超过mask+1个元素。需要注意的是，指针可能是NULL，表示这个缓存bucket没有被占用，另外被占用的bucket可能是不连续的。这个数组可能会随着时间而增长。

#### 元类(Meta Class)
上面我们提到，所有的类自身也是一个对象，我们可以向这个对象发送消息(即调用类方法)。如：
```
NSArray *array = [NSArray array];
```

这个例子中，`+array`消息发送给了`NSArray`类，而这个`NSArray`也是一个对象。既然是对象，那么它也是一个`objc_object`指针，它包含一个指向其类的一个`isa`指针。那么这些就有一个问题了，这个isa指针指向什么呢？为了调用+array方法，这个类的isa指针必须指向一个包含这些类方法的一个`objc_class`结构体。这就引出了`meta-class`的概念

meta-class是一个类对象的类。当我们向一个对象发送消息时，runtime会在这个对象所属的这个类的方法列表中查找方法；而向一个类发送消息时，会在这个类的meta-class的方法列表中查找。

meta-class之所以重要，是因为它存储着一个类的所有类方法。每个类都会有一个单独的meta-class，因为每个类的类方法基本不可能完全相同。
再深入一下，meta-class也是一个类，也可以向它发送一个消息，那么它的isa又是指向什么呢？为了不让这种结构无限延伸下去，Objective-C的设计者让所有的meta-class的isa指向基类的meta-class，以此作为它们的所属类。即，任何NSObject继承体系下的meta-class都使用NSObject的meta-class作为自己的所属类，而基类的meta-class的isa指针是指向它自己。这样就形成了一个完美的闭环。

通过上面的描述，再加上对objc_class结构体中super_class指针的分析，我们就可以描绘出类及相应meta-class类的一个继承体系了，如下图所示：
![](http://img.blog.csdn.net/20150317161312799)

对于NSObject继承体系来说，其实例方法对体系中的所有实例、类和meta-class都是有效的；而类方法对于体系内的所有类和meta-class都是有效的。
讲了这么多，我们还是来写个例子吧：
```
void TestMetaClass(id self, SEL _cmd) {

    NSLog(@"This objcet is %p", self);
    NSLog(@"Class is %@, super class is %@", [self class], [self superclass]);
    
    Class currentClass = [self class];
    for (int i = 0; i < 6; i++) {
        
        
        NSLog(@"Following the isa pointer %d times gives %p, %@", i, currentClass, NSStringFromClass(currentClass));
        if (class_isMetaClass(currentClass)) {
            NSLog(@"----current class is MetaClass");
        }
        
        currentClass = object_getClass(currentClass);
    }
    
    NSLog(@"NSObject's class is %p", [NSObject class]);
    NSLog(@"NSObject's meta class is %p", object_getClass([NSObject class]));
}

#pragma mark -

@implementation Test

- (void)ex_registerClassPair {

    Class newClass = objc_allocateClassPair([NSError class], "TestClass", 0);
    class_addMethod(newClass, @selector(testMetaClass), (IMP)TestMetaClass, "v@:");
    objc_registerClassPair(newClass);

    id instance = [[newClass alloc] initWithDomain:@"some domain" code:0 userInfo:nil];
    [instance performSelector:@selector(testMetaClass)];
}

@end
```

这个例子是在运行时创建了一个NSError的子类TestClass，然后为这个子类添加一个方法testMetaClass，这个方法的实现是TestMetaClass函数。
运行后，打印结果是
```
2015-07-10 14:47:14.326 iTest[1788:31341] This objcet is 0x7fa281d9e290
2015-07-10 14:47:14.326 iTest[1788:31341] Class is TestClass, super class is NSError
2015-07-10 14:47:14.326 iTest[1788:31341] Following the isa pointer 0 times gives 0x7fa281d9e590, TestClass
2015-07-10 14:47:14.327 iTest[1788:31341] Following the isa pointer 1 times gives 0x7fa281d9b890, TestClass
2015-07-10 14:47:14.327 iTest[1788:31341] ----current class is MetaClass
2015-07-10 14:47:14.327 iTest[1788:31341] Following the isa pointer 2 times gives 0x10a0c1198, NSObject
2015-07-10 14:47:14.327 iTest[1788:31341] ----current class is MetaClass
2015-07-10 14:47:14.327 iTest[1788:31341] Following the isa pointer 3 times gives 0x10a0c1198, NSObject
2015-07-10 14:47:14.328 iTest[1788:31341] ----current class is MetaClass
2015-07-10 14:47:14.328 iTest[1788:31341] Following the isa pointer 4 times gives 0x10a0c1198, NSObject
2015-07-10 14:47:14.328 iTest[1788:31341] ----current class is MetaClass
2015-07-10 14:47:14.329 iTest[1788:31341] Following the isa pointer 5 times gives 0x10a0c1198, NSObject
2015-07-10 14:47:14.329 iTest[1788:31341] ----current class is MetaClass
2015-07-10 14:47:14.329 iTest[1788:31341] NSObject's class is 0x10a0c1170
2015-07-10 14:47:14.329 iTest[1788:31341] NSObject's meta class is 0x10a0c1198
```

我们在for循环中，我们通过object_getClass来获取对象的isa，并将其打印出来，依此一直回溯到NSObject的meta-class。可以分析得出:
1.  NSObject 的MetaClass 为RootMetaClass(0x10a0c1198)
2. RootMetaClass的元类为它自身
3. TestClass(0x7fa281d9e590) 的元类为TestClass(0x7fa281d9b890)--此类为MetaClass
4. TestClass(0x7fa281d9b890)的元类为RootMetaClass(0x10a0c1198)

#### 类与对象操作函数
runtime提供了大量的函数来操作类与对象。类的操作方法大部分是以class为前缀的，而对象的操作方法大部分是以objc或object_为前缀。下面我们将根据这些方法的用途来分类讨论这些方法的使用。

#### 类相关操作函数
我们可以回过头去看看objc_class的定义，runtime提供的操作类的方法主要就是针对这个结构体中的各个字段的。下面我们分别介绍这一些的函数。并在最后以实例来演示这些函数的具体用法。

#### 类名(name)
类名操作的函数主要有:
```
const char * class_getName ( Class cls );
```

对于class_getName函数，如果传入的cls为Nil，则返回一个空字符串。


#### 父类(super_class)和元类(meta-class)
父类和元类操作的函数主要有：
```
// 获取类的父类
Class class_getSuperclass ( Class cls );

// 判断给定的Class是否是一个元类
BOOL class_isMetaClass ( Class cls );
```

- class_getSuperclass函数，当cls为Nil或者cls为根类时，返回Nil。不过通常我们可以使用NSObject类的superclass方法来达到同样的目的。
- class_isMetaClass函数，如果是cls是元类，则返回YES；如果否或者传入的cls为Nil，则返回NO。

#### 实例变量大小(instance_size)
实例变量大小操作的函数有：
```
size_t class_getInstanceSize ( Class cls );
```

#### 成员变量(ivars)及属性
在objc_class中，所有的成员变量、属性的信息是放在链表ivars中的。ivars是一个数组，数组中每个元素是指向Ivar(变量信息)的指针。runtime提供了丰富的函数来操作这一字段。大体上可以分为以下几类：
1 成员变量操作函数，主要包含以下函数：
```
// 获取类中指定名称实例成员变量的信息
Ivar class_getInstanceVariable ( Class cls, const char *name );

// 获取类成员变量的信息
Ivar class_getClassVariable ( Class cls, const char *name );

// 添加成员变量
BOOL class_addIvar ( Class cls, const char *name, size_t size, uint8_t alignment, const char *types );

// 获取整个成员变量列表
Ivar * class_copyIvarList ( Class cls, unsigned int *outCount );
```

- class_getInstanceVariable函数，它返回一个指向包含name指定的成员变量信息的objc_ivar结构体的指针(Ivar)。
- class_getClassVariable函数，目前没有找到关于Objective-C中类变量的信息，一般认为Objective-C不支持类变量。注意，返回的列表不包含父类的成员变量和属性。
- Objective-C不支持往已存在的类中添加实例变量，因此不管是系统库提供的提供的类，还是我们自定义的类，都无法动态添加成员变量。但如果我们通过运行时来创建一个类的话，又应该如何给它添加成员变量呢？这时我们就可以使用class_addIvar函数了。不过需要注意的是，这个方法只能在objc_allocateClassPair函数与objc_registerClassPair之间调用。另外，这个类也不能是元类。成员变量的按字节最小对齐量是`1<<alignment`。这取决于ivar的类型和机器的架构。如果变量的类型是指针类型，则传递log2(sizeof(pointer_type))。
-  class_copyIvarList函数，它返回一个指向成员变量信息的数组，数组中每个元素是指向该成员变量信息的objc_ivar结构体的指针。这个数组不包含在父类中声明的变量。outCount指针返回数组的大小。需要注意的是，我们必须使用free()来释放这个数组。

2 属性操作函数，主要包含以下函数：
```
// 获取指定的属性
objc_property_t class_getProperty ( Class cls, const char *name );

// 获取属性列表
objc_property_t * class_copyPropertyList ( Class cls, unsigned int *outCount );

// 为类添加属性
BOOL class_addProperty ( Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount );

// 替换类的属性
void class_replaceProperty ( Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount );
```

这一种方法也是针对ivars来操作，不过只操作那些是属性的值。我们在后面介绍属性时会再遇到这些函数。

3 在MAC OS X系统中，我们可以使用垃圾回收器。runtime提供了几个函数来确定一个对象的内存区域是否可以被垃圾回收器扫描，以处理strong/weak引用。这几个函数定义如下：
```
const uint8_t * class_getIvarLayout ( Class cls );
void class_setIvarLayout ( Class cls, const uint8_t *layout );
const uint8_t * class_getWeakIvarLayout ( Class cls );
void class_setWeakIvarLayout ( Class cls, const uint8_t *layout );
```

但通常情况下，我们不需要去主动调用这些方法；在调用objc_registerClassPair时，会生成合理的布局。在此不详细介绍这些函数

#### 方法(methodLists)
方法操作主要有以下函数：
```
// 添加方法
BOOL class_addMethod ( Class cls, SEL name, IMP imp, const char *types );

// 获取实例方法
Method class_getInstanceMethod ( Class cls, SEL name );

// 获取类方法
Method class_getClassMethod ( Class cls, SEL name );

// 获取所有方法的数组
Method * class_copyMethodList ( Class cls, unsigned int *outCount );

// 替代方法的实现
IMP class_replaceMethod ( Class cls, SEL name, IMP imp, const char *types );

// 返回方法的具体实现
IMP class_getMethodImplementation ( Class cls, SEL name );
IMP class_getMethodImplementation_stret ( Class cls, SEL name );

// 类实例是否响应指定的selector
BOOL class_respondsToSelector ( Class cls, SEL sel );
```

class_addMethod的实现会覆盖父类的方法实现，但不会取代本类中已存在的实现，如果本类中包含一个同名的实现，则函数会返回NO。如果要修改已存在实现，可以使用method_setImplementation。一个Objective-C方法是一个简单的C函数，它至少包含两个参数—self和_cmd。所以，我们的实现函数(IMP参数指向的函数)至少需要两个参数，如下所示：
```
void myMethodIMP(id self, SEL _cmd)
{
    // implementation ....
}
```

与成员变量不同的是，我们可以为类动态添加方法，不管这个类是否已存在。
另外，参数types是一个描述传递给方法的参数类型的字符数组，这就涉及到类型编码，我们将在后面介绍

class_getInstanceMethod、class_getClassMethod函数，与class_copyMethodList不同的是，这两个函数都会去搜索父类的实现

class_copyMethodList函数，返回包含所有实例方法的数组，如果需要获取类方法，则可以使用class_copyMethodList(object_getClass(cls), &count)(一个类的实例方法是定义在元类里面)。该列表不包含父类实现的方法。outCount参数返回方法的个数。在获取到列表后，我们需要使用free()方法来释放它。

class_replaceMethod函数，该函数的行为可以分为两种：如果类中不存在name指定的方法，则类似于class_addMethod函数一样会添加方法；如果类中已存在name指定的方法，则类似于method_setImplementation一样替代原方法的实现。

class_getMethodImplementation函数，该函数在向类实例发送消息时会被调用，并返回一个指向方法实现函数的指针。这个函数会比method_getImplementation(class_getInstanceMethod(cls, name))更快。返回的函数指针可能是一个指向runtime内部的函数，而不一定是方法的实际实现。例如，如果类实例无法响应selector，则返回的函数指针将是运行时消息转发机制的一部分。

class_respondsToSelector函数，我们通常使用NSObject类的respondsToSelector:或instancesRespondToSelector:方法来达到相同目的。

#### 协议(objc_protocol_list)
协议相关的操作包含以下函数：
```
// 添加协议
BOOL class_addProtocol ( Class cls, Protocol *protocol );

// 返回类是否实现指定的协议
BOOL class_conformsToProtocol ( Class cls, Protocol *protocol );

// 返回类实现的协议列表
Protocol * class_copyProtocolList ( Class cls, unsigned int *outCount );
```

- class_conformsToProtocol函数可以使用NSObject类的conformsToProtocol:方法来替代。
- class_copyProtocolList函数返回的是一个数组，在使用后我们需要使用free()手动释放。

#### 版本(version)
版本相关的操作包含以下函数：
```
// 获取版本号
int class_getVersion ( Class cls );

// 设置版本号
void class_setVersion ( Class cls, int version );
```

#### 其它
runtime还提供了两个函数来供CoreFoundation的tool-free bridging使用，即：
```
Class objc_getFutureClass ( const char *name );
void objc_setFutureClass ( Class cls, const char *name );
```

#### 实例(Example)
```
/-----------------------------------------------------------
// MyClass.h

@interface MyClass : NSObject 

@property (nonatomic, strong) NSArray *array;

@property (nonatomic, copy) NSString *string;

- (void)method1;

- (void)method2;

+ (void)classMethod1;

@end

//-----------------------------------------------------------
// MyClass.m

#import "MyClass.h"

@interface MyClass () {
    NSInteger       _instance1;

    NSString    *   _instance2;
}

@property (nonatomic, assign) NSUInteger integer;

- (void)method3WithArg1:(NSInteger)arg1 arg2:(NSString *)arg2;

@end

@implementation MyClass

+ (void)classMethod1 {

}

- (void)method1 {
    NSLog(@"call method method1");
}

- (void)method2 {

}

- (void)method3WithArg1:(NSInteger)arg1 arg2:(NSString *)arg2 {

    NSLog(@"arg1 : %ld, arg2 : %@", arg1, arg2);
}

@end

//-----------------------------------------------------------
// main.h

#import "MyClass.h"
#import "MySubClass.h"
#import 

int main(int argc, const char * argv[]) {
    @autoreleasepool {

        MyClass *myClass = [[MyClass alloc] init];
        unsigned int outCount = 0;

        Class cls = myClass.class;

        // 类名
        NSLog(@"class name: %s", class_getName(cls));

        NSLog(@"==========================================================");

        // 父类
        NSLog(@"super class name: %s", class_getName(class_getSuperclass(cls)));
        NSLog(@"==========================================================");

        // 是否是元类
        NSLog(@"MyClass is %@ a meta-class", (class_isMetaClass(cls) ? @"" : @"not"));
        NSLog(@"==========================================================");

        Class meta_class = objc_getMetaClass(class_getName(cls));
        NSLog(@"%s's meta-class is %s", class_getName(cls), class_getName(meta_class));
        NSLog(@"==========================================================");

        // 变量实例大小
        NSLog(@"instance size: %zu", class_getInstanceSize(cls));
        NSLog(@"==========================================================");

        // 成员变量
        Ivar *ivars = class_copyIvarList(cls, &outCount);
        for (int i = 0; i < outCount; i++) {
            Ivar ivar = ivars[i];
            NSLog(@"instance variable's name: %s at index: %d", ivar_getName(ivar), i);
        }

        free(ivars);

        Ivar string = class_getInstanceVariable(cls, "_string");
        if (string != NULL) {
            NSLog(@"instace variable %s", ivar_getName(string));
        }

        NSLog(@"==========================================================");

        // 属性操作
        objc_property_t * properties = class_copyPropertyList(cls, &outCount);
        for (int i = 0; i < outCount; i++) {
            objc_property_t property = properties[i];
            NSLog(@"property's name: %s", property_getName(property));
        }

        free(properties);

        objc_property_t array = class_getProperty(cls, "array");
        if (array != NULL) {
            NSLog(@"property %s", property_getName(array));
        }

        NSLog(@"==========================================================");

        // 方法操作
        Method *methods = class_copyMethodList(cls, &outCount);
        for (int i = 0; i < outCount; i++) {
            Method method = methods[i];
            NSLog(@"method's signature: %s", method_getName(method));
        }

        free(methods);

        Method method1 = class_getInstanceMethod(cls, @selector(method1));
        if (method1 != NULL) {
            NSLog(@"method %s", method_getName(method1));
        }

        Method classMethod = class_getClassMethod(cls, @selector(classMethod1));
        if (classMethod != NULL) {
            NSLog(@"class method : %s", method_getName(classMethod));
        }

        NSLog(@"MyClass is%@ responsd to selector: method3WithArg1:arg2:", class_respondsToSelector(cls, @selector(method3WithArg1:arg2:)) ? @"" : @" not");

        IMP imp = class_getMethodImplementation(cls, @selector(method1));
        imp();

        NSLog(@"==========================================================");

        // 协议
        Protocol * __unsafe_unretained * protocols = class_copyProtocolList(cls, &outCount);
        Protocol * protocol;
        for (int i = 0; i < outCount; i++) {
            protocol = protocols[i];
            NSLog(@"protocol name: %s", protocol_getName(protocol));
        }

        NSLog(@"MyClass is%@ responsed to protocol %s", class_conformsToProtocol(cls, protocol) ? @"" : @" not", protocol_getName(protocol));

        NSLog(@"==========================================================");
    }
    return 0;
}
```

这段程序的输出如下：
```
2014-10-22 19:41:37.452 RuntimeTest[3189:156810] class name: MyClass
2014-10-22 19:41:37.453 RuntimeTest[3189:156810] ==========================================================
2014-10-22 19:41:37.454 RuntimeTest[3189:156810] super class name: NSObject
2014-10-22 19:41:37.454 RuntimeTest[3189:156810] ==========================================================
2014-10-22 19:41:37.454 RuntimeTest[3189:156810] MyClass is not a meta-class
2014-10-22 19:41:37.454 RuntimeTest[3189:156810] ==========================================================
2014-10-22 19:41:37.454 RuntimeTest[3189:156810] MyClass's meta-class is MyClass
2014-10-22 19:41:37.455 RuntimeTest[3189:156810] ==========================================================
2014-10-22 19:41:37.455 RuntimeTest[3189:156810] instance size: 48
2014-10-22 19:41:37.455 RuntimeTest[3189:156810] ==========================================================
2014-10-22 19:41:37.455 RuntimeTest[3189:156810] instance variable's name: _instance1 at index: 0
2014-10-22 19:41:37.455 RuntimeTest[3189:156810] instance variable's name: _instance2 at index: 1
2014-10-22 19:41:37.455 RuntimeTest[3189:156810] instance variable's name: _array at index: 2
2014-10-22 19:41:37.455 RuntimeTest[3189:156810] instance variable's name: _string at index: 3
2014-10-22 19:41:37.463 RuntimeTest[3189:156810] instance variable's name: _integer at index: 4
2014-10-22 19:41:37.463 RuntimeTest[3189:156810] instace variable _string
2014-10-22 19:41:37.463 RuntimeTest[3189:156810] ==========================================================
2014-10-22 19:41:37.463 RuntimeTest[3189:156810] property's name: array
2014-10-22 19:41:37.463 RuntimeTest[3189:156810] property's name: string
2014-10-22 19:41:37.464 RuntimeTest[3189:156810] property's name: integer
2014-10-22 19:41:37.464 RuntimeTest[3189:156810] property array
2014-10-22 19:41:37.464 RuntimeTest[3189:156810] ==========================================================
2014-10-22 19:41:37.464 RuntimeTest[3189:156810] method's signature: method1
2014-10-22 19:41:37.464 RuntimeTest[3189:156810] method's signature: method2
2014-10-22 19:41:37.464 RuntimeTest[3189:156810] method's signature: method3WithArg1:arg2:
2014-10-22 19:41:37.465 RuntimeTest[3189:156810] method's signature: integer
2014-10-22 19:41:37.465 RuntimeTest[3189:156810] method's signature: setInteger:
2014-10-22 19:41:37.465 RuntimeTest[3189:156810] method's signature: array
2014-10-22 19:41:37.465 RuntimeTest[3189:156810] method's signature: string
2014-10-22 19:41:37.465 RuntimeTest[3189:156810] method's signature: setString:
2014-10-22 19:41:37.465 RuntimeTest[3189:156810] method's signature: setArray:
2014-10-22 19:41:37.466 RuntimeTest[3189:156810] method's signature: .cxx_destruct
2014-10-22 19:41:37.466 RuntimeTest[3189:156810] method method1
2014-10-22 19:41:37.466 RuntimeTest[3189:156810] class method : classMethod1
2014-10-22 19:41:37.466 RuntimeTest[3189:156810] MyClass is responsd to selector: method3WithArg1:arg2:
2014-10-22 19:41:37.467 RuntimeTest[3189:156810] call method method1
2014-10-22 19:41:37.467 RuntimeTest[3189:156810] ==========================================================
2014-10-22 19:41:37.467 RuntimeTest[3189:156810] protocol name: NSCopying
2014-10-22 19:41:37.467 RuntimeTest[3189:156810] protocol name: NSCoding
2014-10-22 19:41:37.467 RuntimeTest[3189:156810] MyClass is responsed to protocol NSCoding
2014-10-22 19:41:37.468 RuntimeTest[3189:156810] ==========================================================
```

#### 动态创建类和对象
runtime的强大之处在于它能在运行时创建类和对象。

##### 动态创建类
动态创建类涉及到以下几个函数：
```
// 创建一个新类和元类
Class objc_allocateClassPair ( Class superclass, const char *name, size_t extraBytes );

// 销毁一个类及其相关联的类
void objc_disposeClassPair ( Class cls );

// 在应用中注册由objc_allocateClassPair创建的类
void objc_registerClassPair ( Class cls );
```

- objc_allocateClassPair函数：如果我们要创建一个根类，则superclass指定为Nil。extraBytes通常指定为0，该参数是分配给类和元类对象尾部的索引ivars的字节数。 为了创建一个新类，我们需要调用objc_allocateClassPair。然后使用诸如class_addMethod，class_addIvar等函数来为新创建的类添加方法、实例变量和属性等。完成这些后，我们需要调用objc_registerClassPair函数来注册类，之后这个新类就可以在程序中使用了。
实例方法和实例变量应该添加到类自身上，而类方法应该添加到类的元类上。

- objc_disposeClassPair函数用于销毁一个类，不过需要注意的是，如果程序运行中还存在类或其子类的实例，则不能调用针对类调用该方法
在前面介绍元类时，我们已经有接触到这几个函数了，在此我们再举个实例来看看这几个函数的使用。
```
Class cls = objc_allocateClassPair(MyClass.class, "MySubClass", 0);
class_addMethod(cls, @selector(submethod1), (IMP)imp_submethod1, "v@:");
class_replaceMethod(cls, @selector(method1), (IMP)imp_submethod1, "v@:");
class_addIvar(cls, "_ivar1", sizeof(NSString *), log(sizeof(NSString *)), "i");

objc_property_attribute_t type = {"T", "@\"NSString\""};
objc_property_attribute_t ownership = { "C", "" };
objc_property_attribute_t backingivar = { "V", "_ivar1"};
objc_property_attribute_t attrs[] = {type, ownership, backingivar};

class_addProperty(cls, "property2", attrs, 3);
objc_registerClassPair(cls);

id instance = [[cls alloc] init];
[instance performSelector:@selector(submethod1)];
[instance performSelector:@selector(method1)];
```
程序的输出如下：
```
2014-10-23 11:35:31.006 RuntimeTest[3800:66152] run sub method 1
2014-10-23 11:35:31.006 RuntimeTest[3800:66152] run sub method 1
```

##### 动态创建对象
动态创建对象的函数如下：
```
// 创建类实例
id class_createInstance ( Class cls, size_t extraBytes );

// 在指定位置创建类实例
id objc_constructInstance ( Class cls, void *bytes );

// 销毁类实例
void * objc_destructInstance ( id obj );
```


class_createInstance函数：创建实例时，会在默认的内存区域为类分配内存。extraBytes参数表示分配的额外字节数。这些额外的字节可用于存储在类定义中所定义的实例变量之外的实例变量。该函数在ARC环境下无法使用。
调用class_createInstance的效果与+alloc方法类似。不过在使用class_createInstance时，我们需要确切的知道我们要用它来做什么。在下面的例子中，我们用NSString来测试一下该函数的实际效果：
```
id theObject = class_createInstance(NSString.class, sizeof(unsigned));
id str1 = [theObject init];

NSLog(@"%@", [str1 class]);

id str2 = [[NSString alloc] initWithString:@"test"];
NSLog(@"%@", [str2 class]);
```

输出结果是：
```
2014-10-23 12:46:50.781 RuntimeTest[4039:89088] NSString
2014-10-23 12:46:50.781 RuntimeTest[4039:89088] __NSCFConstantString
```

可以看到，使用class_createInstance函数获取的是NSString实例，而不是类簇中的默认占位符类_NSCFConstantString
-  objc_constructInstance函数：在指定的位置(bytes)创建类实例。
-  objc_destructInstance函数：销毁一个类的实例，但不会释放并移除任何与其相关的引用。

#### 实例操作函数
实例操作函数主要是针对我们创建的实例对象的一系列操作函数，我们可以使用这组函数来从实例对象中获取我们想要的一些信息，如实例对象中变量的值。这组函数可以分为三小类：
1. 针对整个对象进行操作的函数，这类函数包含
```
// 返回指定对象的一份拷贝
id object_copy ( id obj, size_t size );

// 释放指定对象占用的内存
id object_dispose ( id obj );
```

有这样一种场景，假设我们有类A和类B，且类B是类A的子类。类B通过添加一些额外的属性来扩展类A。现在我们创建了一个A类的实例对象，并希望在运行时将这个对象转换为B类的实例对象，这样可以添加数据到B类的属性中。这种情况下，我们没有办法直接转换，因为B类的实例会比A类的实例更大，没有足够的空间来放置对象。此时，我们就要以使用以上几个函数来处理这种情况，如下代码所示：
```
NSObject *a = [[NSObject alloc] init];
id newB = object_copy(a, class_getInstanceSize(MyClass.class));
object_setClass(newB, MyClass.class);
object_dispose(a);
```

2. 针对对象实例变量进行操作的函数，这类函数包含：
```
// 修改类实例的实例变量的值
Ivar object_setInstanceVariable ( id obj, const char *name, void *value );

// 获取对象实例变量的值
Ivar object_getInstanceVariable ( id obj, const char *name, void **outValue );

// 返回指向给定对象分配的任何额外字节的指针
void * object_getIndexedIvars ( id obj );

// 返回对象中实例变量的值
id object_getIvar ( id obj, Ivar ivar );

// 设置对象中实例变量的值
void object_setIvar ( id obj, Ivar ivar, id value );
```

如果实例变量的Ivar已经知道，那么调用object_getIvar会比object_getInstanceVariable函数快，相同情况下，object_setIvar也比object_setInstanceVariable快。

3. 针对对象的类进行操作的函数，这类函数包含：
```
// 返回给定对象的类名
const char * object_getClassName ( id obj );

// 返回对象的类
Class object_getClass ( id obj );

// 设置对象的类
Class object_setClass ( id obj, Class cls );
```


#### 获取类定义
Objective-C动态运行库会自动注册我们代码中定义的所有的类。我们也可以在运行时创建类定义并使用objc_addClass函数来注册它们。runtime提供了一系列函数来获取类定义相关的信息，这些函数主要包括：
```
// 获取已注册的类定义的列表
int objc_getClassList ( Class *buffer, int bufferCount );

// 创建并返回一个指向所有已注册类的指针列表
Class * objc_copyClassList ( unsigned int *outCount );

// 返回指定类的类定义
Class objc_lookUpClass ( const char *name );
Class objc_getClass ( const char *name );
Class objc_getRequiredClass ( const char *name );

// 返回指定类的元类
Class objc_getMetaClass ( const char *name );
```

objc_getClassList函数：获取已注册的类定义的列表。我们不能假设从该函数中获取的类对象是继承自NSObject体系的，所以在这些类上调用方法是，都应该先检测一下这个方法是否在这个类中实现
```
int numClasses;
Class * classes = NULL;

numClasses = objc_getClassList(NULL, 0);
if (numClasses > 0) {
    classes = malloc(sizeof(Class) * numClasses);
    numClasses = objc_getClassList(classes, numClasses);

    NSLog(@"number of classes: %d", numClasses);

    for (int i = 0; i < numClasses; i++) {

        Class cls = classes[i];
        NSLog(@"class name: %s", class_getName(cls));
    }

    free(classes);
}
```

输出结果如下：
```
2014-10-23 16:20:52.589 RuntimeTest[8437:188589] number of classes: 1282
2014-10-23 16:20:52.589 RuntimeTest[8437:188589] class name: DDTokenRegexp
2014-10-23 16:20:52.590 RuntimeTest[8437:188589] class name: _NSMostCommonKoreanCharsKeySet
2014-10-23 16:20:52.590 RuntimeTest[8437:188589] class name: OS_xpc_dictionary
2014-10-23 16:20:52.590 RuntimeTest[8437:188589] class name: NSFileCoordinator
2014-10-23 16:20:52.590 RuntimeTest[8437:188589] class name: NSAssertionHandler
2014-10-23 16:20:52.590 RuntimeTest[8437:188589] class name: PFUbiquityTransactionLogMigrator
2014-10-23 16:20:52.591 RuntimeTest[8437:188589] class name: NSNotification
2014-10-23 16:20:52.591 RuntimeTest[8437:188589] class name: NSKeyValueNilSetEnumerator
2014-10-23 16:20:52.591 RuntimeTest[8437:188589] class name: OS_tcp_connection_tls_session
2014-10-23 16:20:52.591 RuntimeTest[8437:188589] class name: _PFRoutines
......还有大量输出
```

- 获取类定义的方法有三个：objc_lookUpClass, objc_getClass和objc_getRequiredClass。如果类在运行时未注册，则objc_lookUpClass会返回nil，而objc_getClass会调用类处理回调，并再次确认类是否注册，如果确认未注册，再返回nil。而objc_getRequiredClass函数的操作与objc_getClass相同，只不过如果没有找到类，则会杀死进程。
- objc_getMetaClass函数：如果指定的类没有注册，则该函数会调用类处理回调，并再次确认类是否注册，如果确认未注册，再返回nil。不过，每个类定义都必须有一个有效的元类定义，所以这个函数总是会返回一个元类定义，不管它是否有效。






