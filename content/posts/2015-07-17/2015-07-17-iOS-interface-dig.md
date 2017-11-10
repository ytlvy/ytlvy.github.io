---
author: null
categories:
- IOS
date: '2015-07-17'
description: null
photos: null
tags:
- interface
- dig
title: objc@interface的设计哲学与设计技巧
---

[转自](http://blog.sunnyxx.com/2014/04/13/objc_dig_interface/)

## objc@interface的设计哲学与设计技巧

### 我是前言

学习objc时，尤其是先学过其他编程语言再来看objc时，总会对objc的类声明的关键字`interface`感到有点奇怪，在其它面向对象的语言中通常由class关键字来表示，而interface在java中表示的却大约相当于objc的protocol，这个关键字的区别究竟代表了objc语言的设计者怎样的思想呢，在objc类设计中需要注意哪些问题呢？接下来对这个问题进行一些思考和探究.

### interface?

先来段Wiki:

> In object-oriented programming, a protocol or interface is a common means for unrelated objects to communicate with each other. These are definitions of methods and values which the objects agree upon in order to cooperate.

接口约定了对象间交互的属性和方法，使得对象间无需了解对方就可以协作。
说的洋气点就是解耦嘛，细心点也能发现Wiki中interface和protocol表示了相近的语义。
引用我和项目组架构师讨论有关interface的问题时他的说法:

> interface就是一个object定义的可以被外界影响的方式

说着他指了下旁边桌子上放着的一把伞，说，这把伞我可以打开它，打开这个动作就是它的一个interface，桌子旁边还放着一个盒子，虽然它和伞都放在这张桌子上，但是它们之间永远不会互相影响，所以：

> interface只存在于能互相影响的两者间

### @interface生成了class？

学习`objc`时最早接触的就是怎么写一个类了，从.h中写`@interface`声明类，再从.m中写`@implementation`实现方法，所以，objc中写一个`@interface`就相当于c++中写一个class。但这是真的么？

写个小test验证一下：
有两个类，`Sark`和`Dark`，`Sark`类只有.m文件，其中只写`@implementation`；`Dark`类只有.h头文件，其中只写`@interface`，然后如下测试代码：

```
Class sarkClass = NSClassFromString(@"Sark");
Class darkClass = NSClassFromString(@"Dark");
```

`NSClassFromString`方法调用了runtime方法，根据类名将加载进runtime的这个类找出来，没有这个类就回返回空(Nil)。
结果是`sarkClass`存在，而`darkClass`为空，说明什么？是否说明其实`@implementation`才是真正的Class？
进一步，不止能取到这个没有`@interface`的类，还可以正常调用方法（因为万能的runtime）

如下面的测试代码：
```
Sark *sark = [Sark new];
[sark speak];
```

要是没有@interface的声明，类名，方法名都会报错说找不到，但是可以像下面一样绕一下：
```
Class cls = NSClassFromString(@"Sark");
id obj = [cls performSelector:NSSelectorFromString(@"new")];
[obj performSelector:NSSelectorFromString(@"speak")];
```

其实，从`rewrite`后的`objc`代码可以发现，对于消息的发送，恰恰就是会被处理成类似上面的代码，使用字符串`mapping`出Class，selctor等再使用`objc_msgSend()`进行函数调用，如下面所示：

```
// 经过clang -rewrite-objc 命令重写后的代码
Sark *sark = ((id (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("Sark"), sel_registerName("new"));
((void (*)(id, SEL))(void *)objc_msgSend)((id)sark, sel_registerName("speak"));
```

### 对比@interface和@implementation

`@interface `我们干过的事：

1. 继承
2. 声明协议
3. 定义实例变量（@interface后面加大括号那种）
4. 定义@property
5. 声明方法

`@implementation `我们干过的和可以干的事：

1. 继承
2. 定义实例变量
3. 合成属性（@synthesize和@dynamic）
4. 实现方法（包括协议方法）

在`@implementation`干一些事情用的相对较少，但是是完全合法的，如这样用：

```
@implementation Sark : NSObject {
    NSString *_name;
}
```

通过对比可以发现，`@interface`对objc类结构的合成并无决定性作用，加上无决定性是因为如果没有@interface会丢失一些类自省的原始数据，如属性列表和协议列表，但对于纯粹的对象消息发送并无影响。
所以说，可以得出这么一个结论，objc中`@interface`就是为了给调用者看的，是和调用者的一个protocol，没错，就是__protocol__。

### 对比@interface和@protocol

与其把`@implementation`扯进来不如对比下`@protocol`

我理解objc的`@interface`和`@protocal`间唯一的区别就是是否和一个类型绑定，这让我想起来鸭子类型(Duck typing), [wiki链接](http://zh.wikipedia.org/wiki/%E9%B8%AD%E5%AD%90%E7%B1%BB%E5%9E%8B)

> “当看到一只鸟走起来像鸭子、游泳起来像鸭子、叫起来也像鸭子，那么这只鸟就可以被称为鸭子。”

`Duck type`在objc的体现无疑就是`@protocol`了，我们常用`id<XXXDelegate> delegate`的方式声明一个`delegate`，我们无需care这货到底是什么类型，我们只知道他能干什么就可以work了。同样的功能我也可以使用`XXXDelegate *delegate`的方式来定义，只不过这样的话这个类又需要耦合一个XXXDelegate类型，而这个delegate类是它原本并不需要关心的。

所以说，`@interface`是`@protocol`的强类型升级版。

举个NSObject的例子最合适：
```
@interface NSObject <NSObject> {
    Class isa;
}
```

`NSObject`之所以成为NSObject，绝大多数都是`<NSObject>`协议定义的方法，实体类`@interface`定义的唯一一个变量isa指针，为了继承链和消息传递。
除了`<NSObject>`协议外，NSObject还有很多`Category`来补充它的功能，其实仔细想想，Category更像protocol，一个补充协议，同样不能添加实例变量，但是和`@interface`一样需要与Class绑定。

进一步来讲，自从属性能自动合成变量之后，在头文件@interface中写大括号声明实例变量的情况越来越少（可以参见近几个版本iOS SDK中类头文件里这种写法几乎消失），因此，@interface和@protocol的差别进一步缩小。

---

## 类与接口的设计原则 - 电视和遥控器

我喜欢将`Class`和`interface`的关系比喻成电视+遥控器，那么objc中的消息机制就可以理解成：
用户（caller）通过遥控器（interface）上的按钮（methods）发送红外线（message）来操纵电视（object）
所以，有没有遥控器，电视都在那儿，也就是说，有没有interface，class都是存在的，只是这种存在并没有意义，就好像这个电视没人会打开，没人会用，没人能看，一堆废铁摆在那儿。

![](http://ww4.sinaimg.cn/large/51530583tw1efdy7cw48wj20c108qjru.jpg)

对比简洁的遥控器，一个拥有很多按钮的老式电视遥控器，我们经常会用到的按钮能有几个呢？

![](http://ww4.sinaimg.cn/large/51530583tw1efe08u9hb7j208c0b4jrp.jpg)

所以，在设计一个类的interface的时候，如同在设计遥控器应该有怎样功能的按钮，要从调用者的角度出发，区分边界，应该时刻有以下几点考虑：
1. 这个方法或属性真的属于这个类的职责么？（电视遥控器能遥控空调？）
2. 这个方法或属性真的必须放在.h中（而不是放在.m的类扩展中）么？
3. 调用者必须看文档才能知道这个类该如何使用么？（同一个业务需要调用者按顺序调用多次（而不是将这些细节隐藏，同时提供一个简洁的接口）才行）
4. 调用者是否可以很容易发现类内部的变量和实现方式？（脑补下电视里面一块电路板漏在外面半截- -）
5. …

---

### objc的@interface设计技巧Tips

看过不少代码，从@interface设计上多少就能看出作者的水平，分享下我对于这个问题的一些拙见。

#### 只暴露外部需要看到的
比如，有如下一个类(这个类无意义，主要关注写法) :
```
// Sark.h
@interface SarkViewController : NSObject <NSXMLParserDelegate /*1*/, NSCopying> {
    NSString *_name; // 2
    IBOutlet UITextField *_nameTextField; // 2
}
@property (nonatomic, strong) NSXMLParser *parser; // 3
- (IBAction)nameChangedAction:(id)sender; // 4
@end
```

这个interface出现的问题：

1. 类内部自己使用的协议，如`<NSXMLParserDelegate>`不应该在头文件@interface中声明，而应该在类扩展中声明；公开由外部调用的协议，如`<NSCopying>`则写在这儿是正确的。
2. `实例变量`和`IBOutlet`不应出现在这儿定义，这将类的内部实现暴露了出去，自从属性可以自动合成后，这里就更应该清净了。
3. 内部使用的属性对象不要暴露在外，应该移动到类扩展中。
4. 调用者对IBAction同样不需要关心，那么就不应该放在这儿。

#### 合理分组子功能

- 将相同功能的一组属性或方法写在一起. 使用这个类或者对其进行修改时，一般都是从功能上找，所以把同一功能模块的一组属性或方法写在一块
- 纯操作方法的子功能（无需向类添加变量）使用Category分块
- 在头文件中也可以使用类扩展将interface按功能分区

`Category`里不能添加实例变量，但是类扩展可以，一般都在.m中作为私有interface使用，同样在头文件里作为分区使用，如，ReactiveCocoa中的RACStream.h

#### 避免头文件污染

首先，类实现内部.m文件中使用的其他interface应该在.m文件import，如果也写在header中就会造成对调用者的污染；当interface中出现其他Class或protocol时，可以使用前置声明`@class XXX,` `@protocol XXX`；当模块（一组类）内部间需要有一些定义（如常量、类型）而又不需要模块使用者知道时，使用一个内部头文件在模块中使用。

#### 避免接口过度设计

考虑调用者的使用方便是很必要的，过火了反而增加了复杂度：
```
@interface Sark : NSObject
- (instancetype)init;
- (instancetype)initWithName:(NSString *)name;
- (instancetype)initWithName:(NSString *)name sex:(NSString *)sex;
- (instancetype)initWithName:(NSString *)name sex:(NSString *)sex age:(NSInteger)age;
- (instancetype)initWithName:(NSString *)name sex:(NSString *)sex age:(NSInteger)age friends:(NSArray *)friends;
// 无数多个 //
@end
```

提供了一组这样的方法，调用者可能只能用到其中的一个，那这样倒不如只留一个接口。

#### 避免单例的滥用

单例模式固然好用，但感觉有点过度，将接口设计成单例入口前需要考虑一下：

1. 这个类表达的含义真的只能有一个实例么？（如UIApplication）还是只是为了好调用而已？
2. 这个单例持有的内存一直存在
3. 是否能用类方法代替？
4. 这个单例对象是否能成为另一个单例对象的属性？如果是，应该作为属性

#### 隐藏继承关系中的私有接口
感谢@像条狗在飞在留言中提出的问题，问题大概可以总结为：当子类需要使用父类的一个私有属性（方法）时，需要把这个属性（方法）放到父类的header中，但暴露给子类的同时暴露给了外部调用者，如何解决?

我的方案是：建立一个私有`header`，使用类扩展定义父类需要暴露给子类的属性（方法），然后在各自的.m文件中引用，如：

有Father类和Son类，继承关系，可以考虑建一个如FatherPrivate.h的私有header：
```
// FatherPrivate.h
@interface Father ()
@property (nonatomic, copy) NSString *privateThingSonNeed;
- (void)privateMethodNeedsSonOverride;
@end
```

同时在Father.m和Son.m中同时import这个私有header，这样，Father和Son内部对于定义的属性和方法都是透明的，而对外部是隐藏的（因为两个类的header中都没有import这个私有header）

### 总结

1. @implementation合成了Class，而非@interface，@interface是@protocol的强类型升级版，它们和Category都表示了相近的含义
2. 我们应该善于面向接口编程，划清边界，将类的实现隐藏在调用者所见之外，使主调和被调者之间保持最少知识原则
3. @interface本身就是最好的文档


