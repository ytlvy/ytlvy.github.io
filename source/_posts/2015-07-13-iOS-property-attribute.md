title: iOS property attribute
categories: IOS
tags:
  - iOS
  - property
description:
date: 2015-07-13 21:36:06
author:
photos:
---
## iOS property && ivar && local variable

### default attributes
1. property: atomic assign readwrite
2. ivar && local varibal: strong readwrite non-atomic

> atomic 在实现中通过加入 lock 的方式来保证多线程安全, 但是这只是很简单的一种线程安全, 只局限于此属性的读写.实际应用场景中, 业务的原子性, 是需要自己来实现的.


### when to use copy

1. NSStrings: 为了防止其他地方修改
2. block: 防止自动释放
3. 可变数据类型, 当你想阻止其他拥有者变更数据时.


### 尽量使用 copy 关键字
任何实现了`NSCopying`协议的类型, 都应该尽量采用`copy`. 因为我们定义的属性,在使用时可能是用一个可变的子属性来赋值的, 例如
```
@interface Book : NSObject
 
@property (strong, nonatomic) NSString *title;
 
@end


- (void)stringExample {
 
    NSMutableString *bookTitle = [NSMutableString stringWithString:@"Best book ever"];
 
    Book *book = [[Book alloc] init];
    book.title = bookTitle;
 
    [bookTitle setString:@"Worst book ever"];
 
    NSLog(@"book title %@", book.title);
 
}
```
<!-- more -->
### @dynamic
@synthesize 会自动生成 getter setter方法, @dynamic 只是告诉编译器 getter setter方法 已经定义, 但是不在当前类中(比如在父类中, 或者由 runtime 动态生成).

```
@property (nonatomic, retain) NSButton *someButton;
...
@synthesize someButton;
```


```
@property (nonatomic, retain) IBOutlet NSButton *someButton;
...
@dynamic someButton;
```



