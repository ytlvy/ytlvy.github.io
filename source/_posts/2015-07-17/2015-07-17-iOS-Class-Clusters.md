title: iOS Class Clusters
categories: IOS
tags:
  - Class Clusters
date: 2015-07-17 16:20:40
author:
description:
photos:
---
[转自](http://blog.sunnyxx.com/2014/12/18/class-cluster/)

## 从NSArray看类簇

### Class Clusters
Class Clusters（类簇）是`抽象工厂`模式在iOS下的一种实现，众多常用类，如NSString，NSArray，NSDictionary，NSNumber都运作在这一模式下，它是接口简单性和扩展性的权衡体现，在我们完全不知情的情况下，偷偷隐藏了很多具体的实现类，只暴露出简单的接口。

### NSArray的类簇
虽然[官方文档](https://developer.apple.com/library/ios/documentation/general/conceptual/CocoaEncyclopedia/ClassClusters/ClassClusters.html)中拿`NSNumber`说事儿，但Foundation并没有像图中描述的那样为每个number都弄一个子类，于是研究下NSArray类簇的实现方式。

#### __NSPlacehodlerArray

熟悉这个模式的同学很可能看过下面的测试代码，将原有的alloc+init拆开写：

```
id obj1 = [NSArray alloc]; // __NSPlacehodlerArray *
id obj2 = [NSMutableArray alloc];  // __NSPlacehodlerArray *
id obj3 = [obj1 init];  // __NSArrayI *
id obj4 = [obj2 init];  // __NSArrayM *
```

发现`+ alloc`后并非生成了我们期望的类实例，而是一个`__NSPlacehodlerArray`的中间对象，后面的`- init`或`- initWithXXXXX`消息都是发送给这个中间对象，再由它做工厂，生成真的对象。这里的`__NSArrayI`和`__NSArrayM`分别对应Immutable和Mutable（后面的I和M的意思）

于是顺着思路猜实现，`__NSPlacehodlerArray`必定用某种方式存储了它是由谁alloc出来的这个信息，才能在init的时候知道要创建的是可变数组还是不可变数组

于是乎很开心的去看了下*obj1的内存布局：

![](http://ww4.sinaimg.cn/large/51530583jw1em3doxs660j20l80j6go3.jpg)

下面是32位模拟器中的内存布局（64位太长不好看就临时改32位了- -），第一个箭头是`*obj1`，第二个是`*obj2`

![](http://ww3.sinaimg.cn/mw690/51530583jw1em3dvmuuanj213006maf2.jpg)

我们知道，对象的前4字节（32位下）为isa指针，指向类对象地址，上图所示的`0x0051E768`就是`__NSPlacehodlerArray`类对象地址，可以从lldb下po这个地址来验证。

那么问题来了，这个中间对象并没有储存任何信息诶（除了isa外就都是0了），那它init的时候咋知道该创建什么呢？
经过研究发现，Foundation用了一个很贱的比较静态实例地址方式来实现，伪代码如下：

```
static __NSPlacehodlerArray *GetPlaceholderForNSArray() {
    static __NSPlacehodlerArray *instanceForNSArray;
    if (!instanceForNSArray) {
        instanceForNSArray = [[__NSPlacehodlerArray alloc] init];
    }
    return instanceForNSArray;
}

static __NSPlacehodlerArray *GetPlaceholderForNSMutableArray() {
    static __NSPlacehodlerArray *instanceForNSMutableArray;
    if (!instanceForNSMutableArray) {
        instanceForNSMutableArray = [[__NSPlacehodlerArray alloc] init];
    }
    return instanceForNSMutableArray;
}
// NSArray实现
+ (id)alloc
{
    if (self == [NSArray class]) {
        return GetPlaceholderForNSArray()
    }
}
// NSMutableArray实现
+ (id)alloc
{
    if (self == [NSMutableArray class]) {
        return GetPlaceholderForNSMutableArray()
    }
}
// __NSPlacehodlerArray实现
- (id)init
{
    if (self == GetPlaceholderForNSArray()) {
        self = [[__NSArrayI alloc] init];
    }
    else if (self == GetPlaceholderForNSMutableArray()) {
        self = [[__NSArrayM alloc] init];
    }
    return self;
}
```

Foundation不是开源的，所以上面的代码是猜测的，思路大概就是这样，可以这样验证下：
```
id obj1 = [NSArray alloc]; 
id obj2 = [NSArray alloc];
id obj3 = [NSMutableArray alloc];
id obj4 = [NSMutableArray alloc];
// 1和2地址相同，3和4地址相同，无论多少次都相同，且地址相差16位
```

#### 静态不可变空对象
除此之外，Foundation对`不可变`版本的空数组也做了个小优化：

```
NSArray *arr1 = [[NSArray alloc] init];
NSArray *arr2 = [[NSArray alloc] init];
NSArray *arr3 = @[];
NSArray *arr4 = @[];
NSArray *arr5 = @[@1];
```

上边1-4号都指向了同一个对象，而arr5指向了另一个对象。

若干个不可变的空数组间没有任何特异性，返回一个静态对象也理所应当。
不仅是NSArray，Foundation中如`NSString`, `NSDictionary`, `NSSet`等区分可变和不可变版本的类，空实例都是静态对象（NSString的空实例对象是常量区的@""）

所以也给用这些方法来测试对象内存管理的同学提个醒，很容易意料之外的。






