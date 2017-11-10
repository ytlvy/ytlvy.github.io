---
author: Y.t
categories:
- IOS
date: '2015-06-07'
description: null
tags:
- BLock
- GCD
title: IOS Multithreading && Memory Management
---


## Block
Converting Source Code
```
clang -rewrite-objc file_name_of_the_source_code
```

### 无变量
```
//
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {

    printf("Block\n");
  }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main(){
  void (*blk)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA);

  ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
  return 0;

}
```

<!--more-->

### static变量
```
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;

    int *static_val;

    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc,
    int *_static_val, int flags=0) : static_val(_static_val) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc; 
    }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int *static_val = __cself->static_val;
    (*static_val) *= 3; 
}


```


### __block variable
将原来的变量更改为：Block_byref_val变量，此变量包含一个forwarding影像，用来存放block对其的修改。
```
    //__block variable
    struct __block_impl {
      void *isa;
      int Flags;
      int Reserved;
      void *FuncPtr;
    };

    struct __Block_byref_val_0 {
      void *__isa;
    __Block_byref_val_0 *__forwarding;
     int __flags;
     int __size;
     int val;
    };

    struct __main_block_impl_0 {
      struct __block_impl impl;
      struct __main_block_desc_0* Desc;

      __Block_byref_val_0 *val; // by ref

      __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_val_0 *_val, int flags=0) : val(_val->__forwarding) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
      }
    };

    static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
      __Block_byref_val_0 *val = __cself->val; // bound by ref
    (val->__forwarding->val) = 1;
    }

    static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->val, (void*)src->val, 8/*BLOCK_FIELD_IS_BYREF*/);}

    static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->val, 8/*BLOCK_FIELD_IS_BYREF*/);}

    static struct __main_block_desc_0 {
      size_t reserved;
      size_t Block_size;
      void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
      void (*dispose)(struct __main_block_impl_0*);
    } __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

    struct __main_block_impl_1 {
      struct __block_impl impl;
      struct __main_block_desc_1* Desc;

      __Block_byref_val_0 *val; // by ref

      __main_block_impl_1(void *fp, struct __main_block_desc_1 *desc, __Block_byref_val_0 *_val, int flags=0) : val(_val->__forwarding) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
      }
    };
    static void __main_block_func_1(struct __main_block_impl_1 *__cself) {
      __Block_byref_val_0 *val = __cself->val; // bound by ref
    (val->__forwarding->val) = 2;}

    static void __main_block_copy_1(struct __main_block_impl_1*dst, struct __main_block_impl_1*src) {_Block_object_assign((void*)&dst->val, (void*)src->val, 8/*BLOCK_FIELD_IS_BYREF*/);}

    static void __main_block_dispose_1(struct __main_block_impl_1*src) {_Block_object_dispose((void*)src->val, 8/*BLOCK_FIELD_IS_BYREF*/);}

    static struct __main_block_desc_1 {
      size_t reserved;
      size_t Block_size;
      void (*copy)(struct __main_block_impl_1*, struct __main_block_impl_1*);
      void (*dispose)(struct __main_block_impl_1*);
    } __main_block_desc_1_DATA = { 0, sizeof(struct __main_block_impl_1), __main_block_copy_1, __main_block_dispose_1};

    int main(){

      __attribute__((__blocks__(byref))) __Block_byref_val_0 val = {(void*)0,(__Block_byref_val_0 *)&val, 0, sizeof(__Block_byref_val_0), 10};

      void (*blk)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_val_0 *)&val, 570425344);
      
      void (*blk1)(void) = (void (*)())&__main_block_impl_1((void *)__main_block_func_1, &__main_block_desc_1_DATA, (__Block_byref_val_0 *)&val, 570425344);
      return 0;
    }
    static struct IMAGE_INFO { unsigned version; unsigned flag; } _OBJC_IMAGE_INFO = { 0, 2 };

```
two block share same instance of the __Block_byref_val_0

### Memory Segments for Blocks
1. _NSConcreteStackBlock
2. _NSConcreteGlobalBlock
3. _NSConcreteMallocBlock

|  program area   |
| (.text section) |
|-----------------|
|   data area     |
|   .data section |  <-----  _NSConcreteGlobalBlock
|-----------------|
|                 |    
|       heap      |  <-----  _NSConcreteMallocBlock    
|                 |    
|                 |    
|-----------------|
|      stack      |  <-----  _NSConcreteStackBlock

#### NSConcreteGlobalBlock Class Object
1. when a Block literal is written where a global variable is
```
void (^blk)(void) = ^{printf("Global Block\n");};
int main() {
}
```
Because automatic variables can’t exist where the global variables are declared, capturing never happens

2. when a Block literal is inside a function and doesn’t capture any automatic variables

> Any Block created by another kind of Block literal will be an object of the _NSConcreteStackBlock class, and be stored on the stack.
> 此外所有其他生成的block都是_NSConcreteStackBlock

> A Block, which is stored in the data section like global variables, can be accessed safely via pointers outside any variable scopes. But the other Blocks, which are stored on the stack, will be disposed of after the scope of the Block is left. And __block variables are stored on the stack as well, so the __block variables will be disposed of when the scope is left 
> 全局block， 可以在任何函数域内，安全的进行存取。但是stackBlock会在函数域结束后，被自动释放。为了解决这个问题，Block提供了一种机制，可以将StackBlock复制到heap

#### Block on the Heap
复制到堆上的闭包类型, 变更为NSConcreteMallocBlock. __forwarding变量用来指向闭包,具体的存放位置, stack or heap.

#### Copying Blocks Automatically
在ARC环境下, 系统会自动检测, 并将block复制到heap.
```
typedef int (^blk_t)(int);
blk_t func(int rate)
{
    return ^(int count){return rate * count;};
}
```


#### Coping Blocks Manually
系统不能自动检测的情况
1. When a Block is passed as an argument for methods or functions
   (But if the method or the function copies the argument inside, the caller doesn’t need to copy it manually as)
2. Cocoa Framework methods, the name of which includes “usingBlock”
3. GCD API

you need to copy a Block when you pass it to an NSArray class instance method "initWithObjects"
当你将Block作为参数, 传入数组 'initWithObjects'时,需要手动复制
```
- (id) getBlockArray
{
    int val = 10;
    return [[NSArray alloc] initWithObjects: 
     [^{NSLog(@"blk0:%d", val);} copy],
     [^{NSLog(@"blk1:%d", val);} copy],
      nil];
}
```
尽量多得使用copy对Block无不良影响


#### Memory Segments for __block Variables
1. 当闭包被复制到Heap时, 如果闭包使用了__block变量, 且此变量未被其他block使用, 此变量也会被复制到heap.
2. 如果__block变量被多个闭包使用, 此变量也会被复制到heap. 当二个闭包被复制时, heap中得变量引用指数+1.

#### __forwarding
始终指向最新的Block, 可以对齐进行存取

#### automatic variables of C array type can’t be used in a Block directly
原因是C语言不允许将一个数组变量，赋值另外一个数组变量, 不能编译
```
void func(char a[10])
{
    char b[10] = a;
    printf("%d\n", b[0]); 
}

int main()
{
    char a[10] = {2};
    func(a); 
}
```

#### Capturing Objects
```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  id __strong array;
};
```

#### When is the Block on the stack copied to the heap
􏰀1. When the instance method “copy” is called on the Block
􏰀2. When the Block is returned from a function
􏰀3. When the Block is assigned to a member variable of id or the Block type class, with __strong qualifier
􏰀4. When the Block is passed to a method, including “usingBlock” in the Cocoa Framework, or a Grand Central Dispatch API


#### When You Should Call the “copy” Method
􏰀1. When the Block is returned from a function
􏰀2. When the Block is assigned to a member variable of id or the Block
type class, with a __strong qualifier
􏰀3. When the Block is passed to a method, including “usingBlock” in the Cocoa Framework, or a Grand Central Dispatch API

#### Circular Reference with Blocks
__weak

在不支持weak的情况下
```
__block id tmp = self;
blk_ = ^{
  NSLog(@"self = %@", tmp);
  tmp = nil; 
};
- (void)execBlock {
  blk_();
}

```
如果不执行execBlock, 还是存在循环引用, 要确保执行了execBlock

## GCD

### befor GCD

#### object method
```
- (void) performSelectorInBackground: withObject:
- (void) performSelectorOnMainThread: withObject: waitUntilDone:
```

### 多线程面临的问题
1. race condition
2. dead lock
3. too much threads consumes two much memory

#### 多线程的意义
高交互性， 界面编程中，将耗时的操作放入到其他线程执行，避免影响到主线程界面响应

### GCD 基础
#### Dispatch Queue 派发队列
1. serial dispath queue  顺序派发队列
2. concurrent dispath queue  并发派发队列

和浏览器对同一域名可发起的同时连接数限制一样，可最多同时执行的线程数，也是由系统来决定的

#### 创建队列

```
dispath_queue_t mySerialDispatchQueue = dispatch_queue_create("com.example.gcd.MySerialDispatchQueue", NULL);

dispatch_queue_t myConcurrentDispathQueue = dispatch_queue_create("com.ytlvy.gcd.MyConcurrentDispatchQueue", DISPATCH_QUEUE_CONCURRENT);

dispatch_async(myConcurrentQueue, ^{
  //do staff
  });

dispatch_release(mySerialDispatchQueue);
dispatch_retain(myConcurrentDispatchQueue);

```

当并发队列添加任务后，队列被release，是没有问题的。因为block在添加到队列时，会触发dispatch_retain操作来持有队列，在block结束时，会自动触发dispatch_release来释放队列

#### Main Dispatch Queue / Global Dispatch Queue
可以采用系统方法来获取派发队列。
 NAME                              | Type        | Description  
 ------------------------          | ------------| ----------------
 Main dispatch queue               | Serial      | Executed on the main thread
 Global dispatch queue(Hight)      | Concurrent  | Priority High
 Global dispatch queue(default)    | Concurrent  | Priority Normal
 Global dispatch queue(low)        | Concurrent  | Priority Low
 Global dispatch queue(background) | Concurrent  | Priority Background


```
dispatch_queue_t mainDispatchQueue = dispatch_get_main_queue();
dispatch_queue_t globalDispatchQueueHight = 
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
dispatch_queue_t globalDispatchQueueDefault =
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_queue_t globalDispatchQueueLow = 
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
dispatch_queue_t globalDispatchQueueBackground = 
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
```

dispatch_retain dispatch_release对系统分配的队列不生效。

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0) ^{
   //do staff

   dispatch_async(dispatch_get_main_queue, ^{
    // update gui
    });

  });
```

#### Controlling Dispatch Queue
`dispatch_set_target_queue`设置队列优先级

```
dispatch_queue_t mySerialDispatchQueue = 
    dispatch_create("com.ytlvy.gcd.MySerialDispatchQueue", NULL);
dispatch_queue_t globalDispatchQueueBackGround = 
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
dispatch_set_target_queue(mySerialDispatchQueue, globalDispatchQueueBackground);
```

#### dispatch_after

```
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 3ull * NSEC_PER_SEC);

dispatch_after(time, dispatch_get_main_queue(), ^{

  });
```
"ull" is for "unsigned long long type"

毫秒
```
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull * NSEC_PER_MSEC);
```

```
dispatch_time_t getDispatchTimeByDate(NSDate *date){
  NSTimeInterval interval;
  double second, subsecond;
  struct timespec, time;
  dispatch_time_t milestone;

  interval = [date timeIntervalSince1970];
  subsecond = modf(interval, &second);
  time.tv_sec = second;
  time.tv_nsec = subsecond * NSEC_PER_SEC;
  milestone = dispatch_wailltime(&time, 0);

  return milstone;
}
```

注意，dispatch第二个参数，应该使用`ull` 或者 `double`类型
```
SDate *start = [NSDate date];
dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, 4 * NSEC_PER_SEC);

dispatch_after(popTime, dispatch_get_main_queue(), ^{
  NSLog(@"seconds: %f", [start timeIntervalSinceNow]);
});
// output: seconds: -0.0001

NSDate *start = [NSDate date];
dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, 4.0 * NSEC_PER_SEC);

dispatch_after(popTime, dispatch_get_main_queue(), ^{
  NSLog(@"seconds: %f", [start timeIntervalSinceNow]);
});
// output: seconds: -4.0001
```

#### dispatch group

```
dispatch_queue_t queue = 
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, queue, ^{ NSLog(@"blk0"); });
dispatch_group_async(group, queue, ^{ NSLog(@"blk1"); });
dispatch_group_async(group, queue, ^{ NSLog(@"blk2"); });

dispatch_group_notify(group, dispatch_get_main_queue(), ^{ NSLog(@"done"); });
dispatch_release(group);

```

等所有任务结束，使用派发队列组。
```
dispatch_queue_t queue = 
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, queue, ^{NSLog(@"blk0"); });
dispatch_group_async(group, queue, ^{NSLog(@"blk1"); });
dispatch_group_async(group, queue, ^{NSLog(@"blk2"); });

dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
dispatch_release(group);
```

等待1秒
```
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull * NSEC_PER_SEC);
long result = dispatch_group_wait(group, time);
if(result == 0){
  //finished
}else{
  //some task still running
}
```

检测任务是否完成
```
long result = dispatch_group_wait(group, DISPATCH_TIME_NOW);
```

### dispatch_barrier_async
```
dispatch_queue_t queue = dispatch_queue_create(
      "com.example.gcd.ForBarrier", DISPATCH_QUEUE_CONCURRENT);

dispatch_async(queue, blk0_for_reading);
dispatch_async(queue, blk1_for_reading);
dispatch_async(queue, blk2_for_reading);
dispatch_async(queue, blk3_for_reading);

dispatch_barrier_async(queue, blk_for_writing);

dispatch_async(queue, blk4_for_reading);
dispatch_async(queue, blk5_for_reading);

dispatch_release(queue);
```


### dispatch_sync

dead lock
```
dispatch_queue_t queue = dispatch_get_main_queue();
dispatch_sync(queue, ^{});

//dead lock
dispatch_async(queue, ^{
  dispatch_sync(queue, ^{

    });
  });

//serial queue  dead lock
dispatch_queue_t queue = dispatch_queue_create("com.example", NULL);
dispatch_async(queue, ^{
  dispatch_sync(queue, ^{

    });
  });
```

#### dispatch_apply
与 `dispatch_sync` 和`dispatch_group`相关，可以多次添加同一任务, 并等待任务结束

```
dispatch_queue_t queue =
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_apply(10, queue, ^(size_t index){
  NSLog("%zu", index);
  });
NSLog(@"done"); //last output
```


```
dispatch_queue_t queue = 
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

dispatch_async(queue, ^{

  dispatch_apply([array count], queue, ^(size_t idx){

    });

  //all tasks by dispatch_apply are finished

  dispatch_async(dispatch_get_main_queue(), ^{

    });

  });
```

#### dispatch_suspend && dispatch_resume
```
dispatch_suspend(queue);
dispatch_resume(queue);
```

#### dispatch semaphore
NSMutableArray 是非线程安全的，当多个线程同时更改操作会导致程序崩溃.semaphore是一个更小粒度的多线程控制方法, 通过内部计数来控制线程的准入方式.
semaphore拥有一个内部计数器来模拟标志, 当计数器为0, 线程等待; 非0时,继续执行.
```
dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);

dispatch_remaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
```


```
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull * NSEC_PER_SEC);
long result = dispatch_semaphore_wait(semaphore, time);

if(result == 0){
  //do staff, execute a task
}else{
  //wait 
}
```
当dispatch_semaphore_wait 返回0时, 可以安全执行一个有并发控制的任务. 当任务执行完毕, 需要调用dispatch_semaphore_signal 增加计数

Adding data to NSMutableArray
```
dispatch_queue_t queue = 
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);

NSMutableArray *arr = [NSMutableArray new];
for(int i=0; i< 100000; i++){
  dispatch_async(queue, ^{
      dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);

      /*
       *  因为semaphore的计数大于1. 在disaptch_semaphore_wait返回后,
       *  计数器被设置为0. 因为只有一个线程可以获准进入,所以此时更新
       *  NSMutableArray是安全的.
       */
      [arr addObject:@(i)];

      /*
       * 需要并发控制的任务结束后, 需要调用signal来增加semaphore的计数器,
       * 以便其他等待的线程可以通过dispatch_semaphore_wait, 获得执行权限
       */
      dispatch_semaphore_signal(semaphore);
    });
}

dispatch_release(semaphore);
```

#### dispatch_once
```
static dispatch_once_t pred;
dispatch_once(&pred, ^{

  });
```

### Dispatch I/O
```
dispatch_queue_t pipe_q =
    dispatch_queue_create("PipeQ", NULL);
pipe_channel = dispatch_io_create(DISPATCH_IO_STREAM, fd, pipe_q, ^(int err){
  close(fd);
  });

  out_fd = fdpair[1];
  dispatch_io_set_low_water(pipe_channel, SIZE_MAX);
  dispatch_io_read(pipe_channel, 0, SIZE_MAX, pipe_q, 
    ^(bool done, dispatch_data_t pipedata, int error){
      if(error == 0){
          size_t len = dispatch_data_get_size(pipedata);
          if(len > 0){
            const char *bytes = NULL;
            char *encode;

            dispatch_data_t md = dispatch_data_create_map(
              pipedata, (const void **)&bytes, &len);
            encoded = asl_core_encode_buffer(bytes, len);
            asl_set((aslmsg)merged_msg, ASL_KEY_AUX_DATA, encode);
            free(encoded);

            _asl_send_message(NULL, merged_msg, -1, NULL);
            asl_msg_release(merged_msg);
            dispatch_release(md);
          }
      }

      if(done){
        dispatch_semaphore_signal(semaphore);
        dispatch_relase(pipe_channel);
        dispatch_release(pipe_q);
      }

  });
```
