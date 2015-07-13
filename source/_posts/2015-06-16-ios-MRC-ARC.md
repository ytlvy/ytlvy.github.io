title: 'ios MRC && ARC'
categories: IOS
tags:
  - mrc arc
description:
date: 2015-06-16 09:48:11
author:
photos:
---
## ARC
### MRC
#### MRC内存管理原则
􏰀1. You have ownership of any objects you create.
2.􏰀 You can take ownership of an object using retain.
􏰀3. When no longer needed, you must relinquish ownership of an object you own.
􏰀4. You must not relinquish ownership of an object you don’t own.

Action for Objective-C Object   | Objective-C Method
------------------------------  | -------------------
Create and have ownership of it | alloc/new/copy/mutableCopy group
Take ownership of it            | retian
Relinquish it                   | release
Dispose of it                   | dealloc

#### 调用者持有
Relinquishing Ownership of a Retained Object. 下面的例子将函数将自己持有的obj, 通过return传递给了调用者,由调用者来来持有此变量
```
- (id)allocObject
{
  /*
  - You create an object and have ownership. */
  id obj = [[NSObject alloc] init];
  /*
  - At this moment, this method has ownership of the object. */
  return obj; 
}
```
<!-- more -->

#### 调用者不持有

通过autorelease, 可以让调用者不持有该对象. Autorelease offers a mechanism to relinquish objects properly when the lifetime of the objects has ended.自动释放提供了一种, 当对象生命周期结束时,自动释放的机制.
```
- (id)object
{
  id obj = [[NSObject alloc] init];
  /*
  - At this moment, this method has ownership of the object. */
  [obj autorelease];
  /*
  - The object exists, and you don’t have ownership of it. */
  return obj; 
}

```


#### autorelease
通过autorelease将对象注册到 最近的autoReleasePool中,当pool调用`drain`时, 该对象的release被调用.

#### 不要释放, 非你持有的对象

app crash
```
id obj = [[NSObject alloc] init];
[obj release];
[obj release];
```

```
id obj1 = [obj0 object];
[obj1 release];
```


#### GNUstep -- The alloc Method
```
struct obj_layout {
  NSUInteger retained; 
};
+ (id) alloc {
  int size = sizeof(struct obj_layout) + size_of_the_object; 
  struct obj_layout *p = (struct obj_layout *)calloc(1, size); 
  return (id)(p + 1);
}
```
动态分配地址, 并将第一个字节, 转换为引用计数.

##### GNUstep --  malloc && calloc
malloc 分配的空间没有初始化为0, calloc分配的空间初始化为0; calloc可以分配多个

#### GNUstep --  The retain Method
The alloc method returns a memory block filled with zero containing a struct obj_layout header, which has a variable “retained” to store the number of references. This number is called the reference count

```
id obj = [[NSObject alloc] init]; 
NSLog(@"retainCount=%d", [obj retainCount]); //1
```

```
- (NSUInteger) retainCount {
  return NSExtraRefCount(self) + 1;
}

inline NSUInteger NSExtraRefCount(id anObject){
  return ((struct obj_layout *)anObject)[-1].retained;
}
```

```
- (id) retain {
  NSIncrementExtraRefCount(self);
  return self;
}
inline void NSIncrementExtraRefCount(id anObject) {
  if (((struct obj_layout *)anObject)[-1].retained == UINT_MAX - 1)
      [NSException raise: NSInternalInconsistencyException
                  format: @"NSIncrementExtraRefCount() asked to increment too far"];
 
  ((struct obj_layout *)anObject)[-1].retained++;
}
```

#### GNUstep -- The release Method
```
- (void) release {
  if (NSDecrementExtraRefCountWasZero(self))
    [self dealloc];
}
BOOL NSDecrementExtraRefCountWasZero(id anObject) {
  if (((struct obj_layout *)anObject)[-1].retained == 0) {
    return YES;
  } 
  else {
    ((struct obj_layout *)anObject)[-1].retained--;
    return NO; 
  }
}
```

#### GNUstep -- The dealloc Method
```
- (void) dealloc 
  NSDeallocateObject (self);
}

inline void NSDeallocateObject(id anObject) {
  struct obj_layout *o = &((struct obj_layout *)anObject)[-1];
  free(o); 
}
```

#### GNUstep -- 总结
􏰀1. All Objective-C objects have an integer value called the reference count.
􏰀2. The reference count is incremented by one when one of alloc/new/copy/mutableCopy or retain is called.
􏰀3. It is decremented by one when release is called.
􏰀4. Dealloc is called when the integer counter becomes zero.


#### Apple’s Implementation
```
-retainCount 
  __CFDoExternRefOperation
  CFBasicHashGetCountOfKey

-retain 
  __CFDoExternRefOperation 
  CFBasicHashAddValue

-release 
  __CFDoExternRefOperation 
  CFBasicHashRemoveValue
```

```
int __CFDoExternRefOperation(uintptr_t op, id obj) { 
  CFBasicHashRef table = get hashtable from obj; int count;
  switch (op) {
    case OPERATION_retainCount:
      count = CFBasicHashGetCountOfKey(table, obj); 
      return count;

    case OPERATION_retain: 
      CFBasicHashAddValue(table, obj); 
      return obj;

    case OPERATION_release:
      count = CFBasicHashRemoveValue(table, obj);
      return 0 == count; 
  }
}
```
在GUNstep的实现中, 引用计数存放在每个obj的header部分, 而Apple的实现, 将所有的对象的引用计数存放在一个HashTable中,

GUNstep实现的好处:
- 更少的代码
- 很容易控制生命周期, 因为计数, 本身包含在每个对象中

Apple实现的好处:
- 每个对象不必有header部分, 从而不需要考虑对齐的问题
- 计数统一管理, 每个对象的计数都可以访问到, 便于系统批量处理.
- 当每个对象的内存出问题的时候, 可以通过hashTable来定位到该对象

#### Autorelease
##### Automatic Variables

当一个自动变量, 超出其作用域时, 会自动释放.
```
{
 int a;
}
```

With autorelease, you can use objects in the same manner as automatic variables, meaning that when execution leaves a code block, the “release” method is called on the object automatically. You can control the block itself as well.

The following steps
1. Create an NSAutoreleasePool object.
2. Call “autorelease” to allocated objects.
3. Discard the NSAutoreleasePool object

```
  NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init]; id obj = [[NSObject alloc] init];
  [obj autorelease];
  [pool drain];
```

In the Cocoa Framework, NSAutoreleasePool objects are created, owned, or disposed of all over the place, such as NSRunLoop, which is the main loop of the application 

But when there are too many autoreleased objects, application memory becomes short (Figure 1–14). It happens because the objects still exist until the NSAutoreleasePool object is discarded. A typical example of this is loading and resizing many images. Many autoreleased objects, such as NSData objects for reading files, UImage objects for the data, and resized images exist at the same time.

当有太多autorelease对象时, 程序的可用内存会急剧减少. 因为这些对象会一直存在, 直到pool被结束时.
典型应用场景: 加载或调整大量图片的时候.
```
for (int i = 0; i < numberOfImages; ++i) {
  NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];

  /*
  * Loading images, etc.
  * Too many autoreleased objects exist. 
  */

  [pool drain];
}
```

#### Implementing autorelease

```
[obj autorelease];
```


```
- (id) autorelease {
  [NSAutoreleasePool addObject:self];
}
```

```
+ (void) addObject: (id)anObj {
  NSAutoreleasePool *pool = getting active NSAutoreleasePool; 
  if (pool != nil) {
    [pool addObject:anObj]; 
  } else {
    NSLog(@"autorelease is called without active NSAutoreleasePool.");
  }
}
```

#### Apple’s Implementation of autorelease
runtime/NSObject.mm  #493
```
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
/* equivalent to objc_autoreleasePoolPush() */

id obj = [[NSObject alloc] init];
[obj autorelease];
/* equivalent to objc_autorelease(obj) */

[pool drain];
/* equivalent to objc_autoreleasePoolPop(pool) */
```

### ARC Rules
#### Ownership qualifiers
```
􏰀__strong
􏰀__weak
􏰀__unsafe_unretained
__autoreleasing
```

#### __strong ownership qualifier -- 作用域情况
`__strong`是默认的修饰符. 

```
id obj = [[NSObject alloc] init];
```

等同于
```
id __strong obj = [[NSObject alloc] init];
```

在ARC环境中
```
{/*ARC*/
  id __strong obj = [[NSObject alloc] init];
}
```
当超出变量作用域时, ARC在编译期会自动添加release操作, 以达到和非ARC一致的目的
```
{/*non-ARC*/
  id obj = [[NSObject alloc] init];
  [obj release]; 
}
```

#### Assigning to __strong ownership qualified variables -- 赋值情况

```
{
  /* 持有一个非自己创建的obj */
  id __strong obj = [NSMutableArray array];
} /* 超出作用域, 变量不再使用,  自动添加release*/
/*obj被自动释放*/
```

```
id __strong obj0 = [[NSObject alloc] init]; //objA
id __strong obj1 = [[NSObject alloc] init]; //objB
id __strong obj2 = nil;
obj0 = obj1;  //objA 不再有人持有,自动添加release, objB引用计数+1
obj2 = obj0;  //objB 引用计数+1

obj1 = nil;   //objB引用计数-1
obj0 = nil;   //objB引用计数-1
obj2 = nil;   //objB引用计数-1 当前计数为0 调用dealloc
```
ownership is properly managed not only by variable scope, but also by assignments between variables, which are qualified with __strong. Of course, a __strong qualifier can be used as a member variable of Objective-C class or any argument of methods.
持有关系,不仅可以通过作用域来自动判定, 也可以通过给strong类型的变量赋值来自动判定

```
@interface Test : NSObject
{
  id __strong obj_;
}
- (void)setObject:(id __strong)obj; 
@end

@implementation Test - (id)init
{
    self = [super init];
    return self;
}
- (void)setObject:(id __strong)obj{
  obj_ = obj;
}

```


```
{
  id __strong test = [[Test alloc] init];
  /*test 持有Test obj*/

  [test setObject:[[NSObject alloc] init]]; 
  /*test的属性obj_持有Object*/
}
/*超出作用域, 因为test不再被使用, 自动添加release, test调用release释放Test obj, 因为obj_也讲被销毁 其所持有的Object也会调用release方法 */
/* 变量自动释放完毕*/
```

#### __weak ownership qualifier
避免循环引用.
循环引用的案例: 两个对象同时持有对方, 或者一个对象持有自身.

```
id __weak obj = [[NSObject alloc] init];
```
变量在赋值后, 马上自动释放,因为没有任何变量, 持有新生成的obj.

#### Weak Reference Disappears
当weak变量所指向的变量被释放时, 此weak变量自动指向nil.

```
id __weak obj1 = nil;
{
  id __strong obj0 = [[NSObject alloc] init];
  obj1 = obj0;
  NSLog(@"A: %@", obj1);
}

NSLog(@"B: %@", obj1); // (null)
```

#### __unsafe_unretained ownership qualifier
`__unsafe_unretained` 和`__weak`相似不持有对象.但是当此标示符的变量指向的对象被释放后, 系统不会自动设置该变量为nil.

```
id __unsafe_unretained obj1 = nil;
{
  id __strong obj0 = [[NSObject alloc] init];
  obj1 = obj0;
  NSLog(@"A: %@", obj1);    //<NSObject: 0x753e180>
}
NSLog(@"B: %@", obj1);      //<NSObject: 0x753e180>
/* 此处输出的其实乱指针,因为 生成的对象已经被释放, 但是猜测,由于目前还没有回收该内存, 只是做了标记, 所以暂时指向的地址还是正确的.*/
```

此标示符不建议使用, 除非是为了在ios4等低版本系统中, 为了替换`__weak`.

#### __autoreleasing ownership qualifier

```
  /* non-ARC */
  NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
  id obj = [[NSObject alloc] init];
  [obj autorelease];
  [pool drain];
```

```
/* ARC */
@autoreleasepool {
  id __autoreleasing obj = [[NSObject alloc] init];
}
```

`__autoreleasing`标示符,相当于非ARC环境下调用`autorelease`
但是实际中, 只有非常罕见的情况下才需要设置此标示符.下面解释为什么不需要设置

##### Compiler Cares __autoreleasing Automatically
To obtain an object without creation, you use some methods not in the alloc/new/copy/mutableCopy method group. In this case, the object is automatically registered to the autoreleasepool. It is same as obtaining an autoreleased object. When an object is returned from a method, the compiler checks if the method begins with alloc/new/copy/mutableCopy, and if not, the returned object is automatically registered to the autorelease pool. Exceptionally, any method whose name begins with init, doesn’t register the return value to autoreleasepool. Please see below for more about this new rule: the naming rule for methods related to object creation must be followed
当不是使用alloc/new/copy/mutableCopy之外方法, 来获得的对象时, 系统自动将该对象注册到autoreleasepool中,
```
@autoreleasepool {

  id __strong obj = [NSMutableArray array];
  /*
  * The variable obj is qualified with __strong.
  * Which means, it has ownership of the object.
  * And the object is registered in autoreleasepool,
  * because the compiler decides it by checking the method name. 
  */
}
/*
* Leaving the scope of variable obj, its strong reference disappears.
* The object is released automatically. *
* Leaving @autoreleasepool block,
* all the objects registered in the autoreleasepool are released automatically. *
* The object is discarded because no one has ownership.
*/
```

```
+ (id) array {
  id obj = [[NSMutableArray alloc] init];
  return obj; 
}
```
“id obj” does not have a qualifier. So it is qualified with __strong. When the “return” sentence is executed, the variable scope is left and the strong reference disappears. Therefore the object will be released automatically. Before that, if the compiler detects that the object will be passed to the caller, the object is registered in autoreleasepool

When a variable with a `__weak` qualifier is used, ***the object is always registered in autoreleasepool***
```
id __weak obj1 = obj0; 
NSLog(@"class=%@", [obj1 class]);
```
等同下面
```
id __weak obj1 = obj0;
id __autoreleasing tmp = obj1; 
NSLog(@"class=%@", [tmp class]);
```
weak修饰的变量, 因为不持有对象, 为了安全的释放, weak对象自动注册到autoreleasepool中.


#### weak对象的调用
通过编译代码可以看出, 获取一个weak变量指向的对象, 需要调用`objc_loadWeak`方法, 
```
id objc_loadWeak(id *object) {
  return objc_autorelease(objc_loadWeakRetained(object));
}
```

```
[weakObject doSomething];
```
ARC转换为:
```
Object *strongObject = objc_autorelease(objc_loadWeakRetained(weakObject));
[strongObject doSomething];
```


#### __autoreleasing 默认修饰符

`id *obj`             <==>  `id __autoreleasing *obj`
`NSObject **obj`      <==>  `NSObject * __autoreleasing *obj`
Any pointers to ‘id’ or object types are qualified with __autoreleasing as default.

#### Returning a Result as the Argument
NSError作为参数, 返回结果时. 满足非通过alloc/new/copy/mutableCopy生成的对象, 不持有该对象原则.所以此对象需要自动注册到autoreleasepoool中, 来进行自动释放.
```
- (BOOL) performOperationWithError:(NSError * __autoreleasing *)error {
  /* Error occurred. Set errorCode */
  return NO; 
}
```
By assigning to *error, which is NSError * __autoreleasing * type, an object can be passed to its caller after being registered in autoreleasepool.

```
NSError *error = nil;       //strong
NSError **pError = &error;  //weak   编译报错,不能将strong对象 赋值给weak
```

```
NSError *error = nil;
NSError * __strong *pError = &error;

NSError __weak *error = nil; 
NSError * __weak *pError = &error;

NSError __unsafe_unretained *unsafeError = nil;
NSError * __unsafe_unretained *pUnsafeError = &unsafeError;
```

NSRunLoop has autoreleasepool to release all the objects once in each loop. 
`_objc_autoreleasePoolPrint();`打印注册的所有对象

#### Rules
- Forget about using retain, release, retainCount, and autorelease.
- Forget about using NSAllocateObject and NSDeallocateObject.
- Follow the naming rule for methods related to object creation. (alloc/new/copy/mutablecopy/init)
- Forget about calling dealloc explicitly.
- Use @autoreleasepool instead of NSAutoreleasePool.
- Forget about using Zone (NSZone).
- Object type variables can’t be members of struct or union in C language.
- 'id' and 'void*' have to be cast explicitly.

in many cases, dealloc is a suitable place to remove the object from delegate or observers.
```
- (void) dealloc {
  /*
  - Write here to be disposed of properly.
  */
  [[NSNotificationCenter defaultCenter] removeObserver:self];
}
```
ARC环境下不需显示调用`[super dealloc]`了.

#### Object Type Variables Cannot Be Members of struct or union in C Language
```
struct Data {
  NSMutableArray *array;   //error
};
```
因为ARC环境下, 编译器需要掌握对象的生命周期, 来进行内存管理,但是C结构体, 没有这些信息,没有被ARC所覆盖.可以通过`__unsafe_unretained`修饰符暂时停止该对象的ARC管理来解决
```
struct Data {
  NSMutableArray __unsafe_unretained *array; 
};
```

#### ‘id’ and ‘void*’ Have to Be Cast Explicitly

##### __bridge cast
```
id obj = [[NSObject alloc] init]; 
void *p = (__bridge void *)obj;    //more dangerous than an __unsafe_unretained, 需要手动管理内存
id o = (__bridge id)p;
```

##### __bridge_retained cast
__bridge_retained cast works as if the assigned variable has ownership of the object.
```
id obj = [[NSObject alloc] init];
void *p = (__bridge_retained void *)obj;
```
__bridge_retained的名称是因为, (void *)为C语言实现,不再ARC的管理范围, 所以该次赋值, 需要将对象的计数器+1, 以实现转换后的变量, 对此对象的持有操作.

Non ARC
```
id obj = [[NSObject alloc] init];
void *p = obj; 
[(id)p retain];
```
在非ARC环境下, 变量在转换后, 需要执行retain操作.

```
void *p = 0;
{
  id obj = [[NSObject alloc] init];
  p = (__bridge_retained void *)obj; 
}
NSLog(@"class=%@", [(__bridge id)p class]);
```
obj在离开作用域后, 由于不再使用,自动调用release释放, 而此时由于p仍然持有对象, 所以对象没有被销毁.

```
/* non-ARC */ void *p = 0;
{
  id obj = [[NSObject alloc] init];    /* [obj retainCount] -> 1 */
  p = [obj retain];                    /* [obj retainCount] -> 2 */
  [obj release];                       /* [obj retainCount] -> 1 */
}
NSLog(@"class=%@", [(__bridge id)p class]);

```

#### __bridge_transfer cast
```
id obj = (__bridge_transfer id)p;
```

```
/* non-ARC */
id obj = (id)p; 
[obj retain]; 
[(id)p release];
```
As `__bridge_retained` cast is replaced with `retain`, `__bridge_transfer` cast is replaced with `release`. 

The variable obj is retained because it is qualified with __strong. With these two casts, you can create, own, and release any objects without using ‘id’ or object type variables. But it is not recommended

```
void *p = (__bridge_retained void *)[[NSObject alloc] init]; 
NSLog(@"class=%@", [(__bridge id)p class]); 
(void)(__bridge_transfer id)p;
```

```
/* non-ARC */
id p = [[NSObject alloc] init];
NSLog(@"class=%@", [p class]);
[p release];
```

#### CoreFoundation Object && Objective-C Object
```
CFTypeRef CFBridgingRetain(id X) {
  return (__bridge_retained CFTypeRef)X; 
}
id CFBridgingRelease(CFTypeRef X) {
  return (__bridge_transfer id)X; 
}
```


#### CFBridgingRetain function
```
CFMutableArrayRef cfObject = NULL;
{
  id obj = [[NSMutableArray alloc] init];
  cfObject = CFBridgingRetain(obj);
  CFShow(cfObject);
  printf("retain count = %d\n", CFGetRetainCount(cfObject));
}
printf("retain count after the scope = %d\n", CFGetRetainCount(cfObject)); 
CFRelease(cfObject);
```

换成`__bridge`会造成空指针

```
CFMutableArrayRef cfObject = NULL;
{
  id obj = [[NSMutableArray alloc] init];
  /*variable obj has a strong reference to the object */

  cfObject = (__bridge CFMutableArrayRef)obj; 
  CFShow(cfObject);
  printf("retain count = %d\n", CFGetRetainCount(cfObject));
  /*
   * __bridge cast does not touch ownership status.
   * Reference count is one because of variable obj's strong reference.
   */ 
}
/*
 * Leaving the scope of variable obj, its strong reference disappears.
 * The object is released automatically.
 * Because no one has ownership, the object is discarded. 
 */

/* From here, any access to the object is invalid! (dangling pointer) */
printf("retain count after the scope = %d\n", CFGetRetainCount(cfObject)); 
CFRelease(cfObject);
```

#### CFBridgingRelease function
```
{
  CFMutableArrayRef cfObject =
      CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
  printf("retain count = %d\n", CFGetRetainCount(cfObject));

  /*
   * The object is created with ownership by Core Foundation Framework API. 
   * The retain count is one.
   */

  id obj = CFBridgingRelease(cfObject);
  /*
  * By assignment after CFBridgingRelease,
  * variable obj has a strong reference and then * the object is released by CFRelease.
  */


  printf("retain count after the cast = %d\n", CFGetRetainCount(cfObject));

  /*
  * Only the variable obj has a strong reference to
  * the object, so the retain count is one. *
  * And, after being cast by CFBridgingRelease,
  * pointer stored in variable cfObject is still valid. 
  */

  NSLog(@"class=%@", obj);
}
/*
* Leaving the scope of variable obj, its strong reference disappears.
* The object is released automatically.
* Because no one has ownership, the object is discarded. 
*/

```

如果换成`__bridge`会造成内存泄露

#### Property

Property modifier       | Ownership qualifier
------------------------| --------------------------
assign                  | __unsafe_unretained
copy                    | __strong (note: new copied object is assigned.)
retain                  | __strong
strong                  | __strong
unsafe_unretained       | __unsafe_unretained
weak                    | __weak

#### Array
By the way, any variables qualified with __strong, __weak, or __autoreleasing other than __unsafe_unretained, are initialized with nil.
```
{
  id objs[2];
  objs[0] = [[NSObject alloc] init];
  objs[1] = [NSMutableArray array]; 
}
```
When the control flow leaves the scope of the array, all the variables that have strong references in the array disappear. The assigned objects are released automatically. It is identical to variables not in arrays

```
NSObject * __strong *array = nil;
array = (id __strong *)calloc(entries, sizeof(id));

//or
array = (id __strong *)malloc(entries * sizeof(id)); 
memset(array, 0, entries * sizeof(id));
```
C语音创建的数组需要手动释放
```
for (NSUInteger i = 0; i < entries; ++i){
  array[i] = nil; 
}
free(array);
```


### ARC Implementation
> Automatic Reference Counting (ARC) in Objective-C makes memory management the job of the compiler

but the truth is, ARC isn’t the only job of the compiler. The objective-C runtime needs to help as well

#### __strong ownership qualifier
```
{
  id __strong obj = [[NSObject alloc] init];
}
```

pseudo code by the compiler
```
id obj = objc_msgSend(NSObject, @selector(alloc)); 
objc_msgSend(obj, @selector(init)); 
objc_release(obj);
```

#### Calling the array method
```
{
  id __strong obj = [NSMutableArray array];
}

```

pseudo code by the compiler
```
id obj = objc_msgSend(NSMutableArray, @selector(array)); objc_retainAutoreleasedReturnValue(obj); 
objc_release(obj);
```

`objc_retainAutoreleasedReturnValue` function is for performance optimization. It is inserted because the NSMutableArray class method array is not in the alloc/new/copy/mutableCopy method group. The compiler inserts this function every time just after the invocation of a method if the method is not in the group. As the name suggests, it retains an object returned from a method or function ***after the object is added in autorelease pool***.

#### Inside the array Method
```
+ (id) array {
  return [[NSMutableArray alloc] init];
}
```

pseudo code
```
+ (id) array {
  id obj = objc_msgSend(NSMutableArray, @selector(alloc)); 
  objc_msgSend(obj, @selector(init));
  return objc_autoreleaseReturnValue(obj);
}
```
in reality, `objc_autoreleaseReturnValue` doesn’t register it to autorelease pool all the time. 如果之前的代码已经调用了`objc_retainAutoreleasedReturnValue`,该函数将不再调用,因为此对象已经注册到pool中.

#### __weak ownership qualifier
weak特性:

- **Nil** is assigned to any variables qualified with __weak when referencing object is discarded.
- When an object is accessed through a **__weak** qualified variable, the object is ***added to the autorelease pool***.

```
{
id __weak obj1 = obj; 
}
```

pseudo code
```
id obj1;
objc_initWeak(&obj1, obj); 
objc_destroyWeak(&obj1);
```

`objc_initWeak` 实现

```
obj1 = 0; 
objc_storeWeak(&obj1, obj);
```

`objc_destroyWeak`

```
id obj1;
obj1 = 0;
objc_storeWeak(&obj1, obj); 
objc_storeWeak(&obj1, 0);
```
objc_storeWeak function registers a `key-value` to a `table`, called a `weak table`. The key is the second argument, the address of the object to be assigned. The value is the first argument, the address of a variable that qualified with __weak. **If the second argument is zero, the entry is removed from the table.**

The weak table is implemented as a hash table as a reference count table (see Chapter 1, Section “The Implementation by Apple”). With that, variables qualified with __weak can be searched from a disposing object with reasonable performance. When the function is called with the same object for key, multiple __weak qualified variables will be registered for the same object.

#### Looking Under the Hood When an Object Is Discarded 对象销毁
对象销毁流程

1. objc_release.
2. dealloc is called because retain count becomes zero
3. _objc_rootDealloc
4. object_dispose
5. objc_destructInstance
6. objc_clear_deallocating

objc_clear_deallocating 作用
1. From the weak table, get an entry of which the key is the object to be discarded.
2. Set nil to all the __weak ownership qualified variables in the entry
3. Remove the entry from the table
4. For the object to be disposed of, remove its key from the reference table

#### Assigning a Newly Created Object

```
{
  id __weak obj = [[NSObject alloc] init];
}
```

pseudo code
```
id obj;
id tmp = objc_msgSend(NSObject, @selector(alloc)); 
objc_msgSend(tmp, @selector(init)); 
objc_initWeak(&obj, tmp);
objc_release(tmp);
objc_destroyWeak(&object);
```

#### Immediate Disposal of Objects
声明后马上释放的对象, 不影响方法的调用
```
(void)[[[NSObject alloc] init] hash];
```

```
id tmp = objc_msgSend(NSObject, @selector(alloc)); 
objc_msgSend(tmp, @selector(init)); 
objc_msgSend(tmp, @selector(hash)); 
objc_release(tmp);
```

#### Adding to autorelease pool Automatically
when an object is accessed through a __weak qualified variable, the object has been added to the autorelease pool.
```
{
  id __weak obj1 = obj;
  NSLog(@"%@", obj1); 
}
```

pseudo code by the compiler
```
id obj1;
objc_initWeak(&obj1, obj);
id tmp = objc_loadWeakRetained(&obj1); 
objc_autorelease(tmp);
NSLog(@"%@", tmp); 
objc_destroyWeak(&obj1);
```

`objc_loadWeakRetained` and `objc_autorelease` function calls are newly inserted.
- objc_loadWeakRetained function retains the object referenced by the variable qualified with __weak
- objc_autorelease function adds the object to the autorelease pool.
(自我猜测) 
1. weak只是一个标示, 不能针对weak的变量直接进行方法调用
2. 基于线程安全的考虑, 在weak变量在调用方法时, 先将对象retain, 以保证对象不被其他线程释放.

#### weak变量使用优化
***由于weak对象每次调用都需要注册到autoreleasepool中, 如果此类变量太多会加重pool的负担, 所以在使用时,应将weak变量, 重新赋值到一个strong变量中来操作***
```
{
id __weak o = obj; 
NSLog(@"1 %@", o); 
NSLog(@"2 %@", o);
NSLog(@"3 %@", o); 
NSLog(@"4 %@", o); 
NSLog(@"5 %@", o);
}
```

#### 不支持weak的情况
第一种旧系统不支持, 第二种情况某些类由于重写了retain/release, 并拥有独立的引用计数机制.这些类有`objc_arc_weak_reference_unavailable`标示.

```
- (BOOL)allowsWeakReference; 
- (BOOL)retainWeakReference;
```

```
@interface MyObject : NSObject {
  NSUInteger count;
}
@end

@implementation MyObject - (id)init {
  self = [super init];
  return self;
}
- (BOOL)retainWeakReference {
    if (++count > 3)
      return NO;
    return [super retainWeakReference];
} 
@end
```

```
{
id __strong obj = [[MyObject alloc] init];
id __weak o = obj;
NSLog(@"1 %@", o);   // <MyObject: 0x753e180>
NSLog(@"2 %@", o);   // <MyObject: 0x753e180>
NSLog(@"3 %@", o);   // <MyObject: 0x753e180>
NSLog(@"4 %@", o);   // (null)
NSLog(@"5 %@", o);   // (null)
}
```

#### __autoreleasing ownership qualifier

```
@autoreleasepool {
id __autoreleasing obj = [[NSObject alloc] init]; 
}
```

```
/* pseudo code by the compiler */
id pool = objc_autoreleasePoolPush();
id obj = objc_msgSend(NSObject, @selector(alloc)); 
objc_msgSend(obj, @selector(init)); 
objc_autorelease(obj); 
objc_autoreleasePoolPop(pool);
```

非持有变量
```
@autoreleasepool {
  id __autoreleasing obj = [NSMutableArray array]; 
}
```

```
/* pseudo code by the compiler */
id pool = objc_autoreleasePoolPush();
id obj = objc_msgSend(NSMutableArray, @selector(array)); objc_retainAutoreleasedReturnValue(obj); 
objc_autorelease(obj);
objc_autoreleasePoolPop(pool);
```


#### __unsafe_unretained ownership qualifier
 Unlike the other ownership qualifiers, the compiler does nothing special for the qualifier. It just works as an assignment in C language. ARC不对其进行操作优化.


#### Reference Count
可以通过`_objc_rootRetainCount` 来获取当前计数
```
{
id __strong obj = [[NSObject alloc] init];
NSLog(@"retain count = %d", _objc_rootRetainCount(obj)); 
}
```


```

{
id __strong obj = [[NSObject alloc] init];
id __weak o = obj;
NSLog(@"retain count = %d", _objc_rootRetainCount(obj));  // 1
}
```


```
@autoreleasepool {
  id __strong obj = [[NSObject alloc] init];
  id __autoreleasing o = obj;
  NSLog(@"retain count = %d", _objc_rootRetainCount(obj));  //2
}
```


```
{
id __strong obj = [[NSObject alloc] init]; 
@autoreleasepool {
  id __autoreleasing o = obj;
  NSLog(@"retain count = %d in @autoreleasepool", _objc_rootRetainCount(obj)); //2
}
  NSLog(@"retain count = %d", _objc_rootRetainCount(obj)); //1
}

```

```
@autoreleasepool {
id __strong obj = [[NSObject alloc] init];
_objc_autoreleasePoolPrint();
id __weak o = obj;
NSLog(@"before using __weak: retain count = %d", _objc_rootRetainCount(obj)); NSLog(@"class = %@", [o class]);
NSLog(@"after using __weak: retain count = %d", _objc_rootRetainCount(obj)); _objc_autoreleasePoolPrint();
}
```
