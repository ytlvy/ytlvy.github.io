---
layout: post
title: "ios 内存管理"
date: 2015-05-27 15:47:24 +0800
comments: true
categories: IOS
---
#### __autoreleasing关键字
1. ARC下使用__autoreleasing修饰符修饰的变量会被注册到autoreleasepool中，和非ARC下调用autorelease方法的效果相同
2. 当方法名不是alloc/new/copy/mutableCopy时，使用return返回的对象强引用会被自动注册到autoreleasepool
```objc
    + (id)array  {  
      id obj = [[NSArray alloc] init];  
      return obj;  
    }
```
上述代码中，因为没有指定所有权修饰符，id obj就是默认的id __strong obj，即obj是强引用，这时，obj就会被添加到autoreleasepool中

<!--more-->

3. 访问附有__weak修饰符的变量时，该变量会被自动注册到autoreleasepool中，比如
```
id __weak obj1 = obj0;  
NSLog("class=%@", [obj1 class]);  
  
//等价于以下代码  
  
id __weak obj1 = obj0;  
id __autoreleasing tmp = obj1;  
NSLog("class=%@", [tmp class]);  
```
这种等价是编译器自动做的转换，原因是__weak修饰符支持有对象的弱引用，在访问引用对象的过程中，该对象可能被释放。而如果将该对象加入到autoreleasepool中，在pool被释放之前，tmp对该对象的引用都是有效的

4. _objc_autoreleasePoolPrint()

#### __weak关键字
实际上，objc_storeWeak函数会把第二个参数的对象的地址作为key，并将第一个参数（__weak关键字修饰的指针的地址）作为值，注册到weak表中。如果第二个参数为0（说明对应的对象被释放了），则将weak表中将整个key-value键值对删除，这就是__weak关键字的核心思想！

weak表和引用计数表类似，都是通过hash表实现的。如果使用weak表，将被释放的对象地址作为key去检索，就能很高效的获取对应的指向该对象的类型为__weak的指针变量的地址。同时很容易理解，一个对象可能有多个__weak指针指向，因此一个对象地址key可能对应多个值

在调用对象的release方法时，会在其中一步调用objc_clear_deallocating函数，该函数会执行以下操作
 * 以当前对象的地址作为key，从weak表中获取对应的值----指向该对象的__weak类型的指针变量；
 * 将取到的所有指针变量的值赋值为nil
 * 从weak表中删除该key对应的整条记录

#### arc rules
1. You have ownership of any objects you create
2. You can take ownership of an object using retain
3. When you no longer need it, you must relinquish ownership of an
object of which you have ownership
4. You must not relinquish ownership of an object of which you don’t have ownership

#### 注意问题
1. __weake 与alloc copy mutablecopy new 不能一起用
```
    id __weake obj = [[NSObject alloc] init];
    //obj为nil，因为weak不能持有对象，所以生成后马上释放
    id __unsafe_unretained obj = [[NSObject alloc] init];
    //obj is nil
```
2. NSAutoReleasePool 对象不能调用 autorelease 方法  因为NSAutoReleasePool类重载了该方法
3. arc 使用autoreleasepool
```
    @autoreleasepool{
        id __autoreleasing obj2;
        obj2 = obj;// equivalent to [obj autorelease];
    }
```

4. When a variable with a __weak qualifier is used, the object is always registered in autoreleasepool
5. 默认标志
```
    NSError *error = nil; //__strong
    NSError **pError = &error;//error because __autoreleasing
    NSError * __strong *pError = &error;//true
```
6. arc可以用dealloc， 不需要写[super dealloc]
7. ARC forbids Objective-C objs in structs or unions ; it is impossible for the compiler to perform memory management for C struct
you can do this way
```
struct Data {
NSMutableArray __unsafe_unretained *array;
};
```
| arc       |    mrc   |
| :-------- | --------:|
| id obj    | id __strong obj |
| id *obj   | id __autoreleasing *obj |
| NSObject **obj | NSObject __autoreleasing **obj | 

#### __bridge cast
    id obj = [[NSObject alloc] init]; 
    void *p = (__bridge void *)obj; 
    id o = (__bridge id)p

#### __bridge_retained cast
__bridge_retained cast works as if the assigned variable has ownership of the object；__bridge_retained cast is replaced with retain

    id obj = [[NSObject alloc] init];
    void *p = (__bridge_retained void *)obj;


#### __bridge_transfer cast
__bridge_transfer cast is replaced with release

    id obj = (__bridge_transfer id)p；
    
    /* non-ARC */
    id obj = (id)p; 
    [obj retain]; 
    [(id)p release];

#### CFBridging
    CFMutableArrayRef cfObject = NULL; {
    id obj = [[NSMutableArray alloc] init];
    cfObject = CFBridgingRetain(obj);
    CFShow(cfObject);
    printf("retain count = %d\n", CFGetRetainCount(cfObject));
    }
    printf("retain count after the scope = %d\n", CFGetRetainCount(cfObject)); CFRelease(cfObject);

