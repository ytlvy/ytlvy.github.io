---
author: null
categories:
- IOS
date: '2015-07-17'
description: null
photos: null
tags:
- dig
- kvo
title: iOS kvo dig
---

[转自](http://blog.sunnyxx.com/2014/03/09/objc_kvo_secret/)

## objc kvo简单探索

`KVO(Key Value Observing)`，是观察者模式在Foundation中的实现

### KVO的原理
简而言之就是：

1. 当一个object有观察者时，动态创建这个object的类的子类
2. 对于每个被观察的property，重写其set方法
3. 在重写的set方法中调用- willChangeValueForKey:和- didChangeValueForKey:通知观察者
4. 当一个property没有观察者时，删除重写的方法
5. 当没有observer观察任何一个property时，删除动态创建的子类

空说无凭，简单验证下。

```
@interface Sark : NSObject
@property (nonatomic, copy) NSString *name;
@end

@implementation Sark
@end

Sark *sark = [Sark new];
// breakpoint 1
[sark addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew context:nil];
// breakpoint 2
sark.name = @"萨萨萨";
[sark removeObserver:self forKeyPath:@"name"];
// breakpoint 3
```

断住后分别使用- class和object_getClass()打出sark对象的Class和真实的Class

```
// breakpoint 1
(lldb) po sark.class
Sark
(lldb) po object_getClass(sark)
Sark

// breakpoint 2
(lldb) po sark.class
Sark
(lldb) po object_getClass(sark)
NSKVONotifying_Sark

// breakpoint 3
(lldb) po sark.class
Sark
(lldb) po object_getClass(sark)
Sark
```

上面的结果说明，在sark对象被观察时，framework使用runtime动态创建了一个Sark类的子类`NSKVONotifying_Sark`
而且为了隐藏这个行为，`NSKVONotifying_Sark`重写了`- class`方法返回之前的类，就好像什么也没发生过一样
但是使用`object_getClass()`时就暴露了，因为这个方法返回的是这个对象的isa指针，这个指针指向的一定是个这个对象的类对象

---

然后来偷窥一下这个动态类实现的方法，这里请出一个NSObject的扩展NSObject+DLIntrospection，它封装了打印一个类的方法、属性、协议等常用调试方法，一目了然。
```
@interface NSObject (DLIntrospection)
+ (NSArray *)classes;
+ (NSArray *)properties;
+ (NSArray *)instanceVariables;
+ (NSArray *)classMethods;
+ (NSArray *)instanceMethods;

+ (NSArray *)protocols;
+ (NSDictionary *)descriptionForProtocol:(Protocol *)proto;

+ (NSString *)parentClassHierarchy;
@end
```

然后继续在刚才的断点处调试：

```
// breakpoint 1
(lldb) po [object_getClass(sark) instanceMethods]
<__NSArrayI 0x8e9aa00>(
- (void)setName:(id)arg0 ,
- (void).cxx_destruct,
- (id)name
)
// breakpoint 2
(lldb) po [object_getClass(sark) instanceMethods]
<__NSArrayI 0x8d55870>(
- (void)setName:(id)arg0 ,
- (class)class,
- (void)dealloc,
- (BOOL)_isKVOA
)
// breakpoint 3
(lldb) po [object_getClass(sark) instanceMethods]
<__NSArrayI 0x8e9cff0>(
- (void)setName:(id)arg0 ,
- (void).cxx_destruct,
- (id)name
)
```

从上面breakpoint2的打印可以看出，动态类重写了4个方法：

1. `- setName:` 最主要的重写方法，set值时调用通知函数
2. `- class` 隐藏自己必备啊，返回原来类的class
3. `- dealloc` 做清理犯罪现场工作
4. `- _isKVOA` 这就是内部使用的标示了，判断这个类有没被KVO动态生成子类

---

接下来验证一下KVO重写set方法后是否调用了`- willChangeValueForKey:`和`- didChangeValueForKey:`
最直接的验证方法就是在Sark类中重写这两个方法：

```
@implementation Sark

- (void)willChangeValueForKey:(NSString *)key
{
    NSLog(@"%@", NSStringFromSelector(_cmd));
    [super willChangeValueForKey:key];
}

- (void)didChangeValueForKey:(NSString *)key
{
    NSLog(@"%@", NSStringFromSelector(_cmd));
    [super didChangeValueForKey:key];
}

@end
```




