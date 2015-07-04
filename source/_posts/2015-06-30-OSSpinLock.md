title: OSSpinLock
categories: IOS
tags:
  - IOS 
  - LOCK
  - OSSpinLock
description: IOS LOCk 
date: 2015-06-30 09:10:44
author:
photos:
---
## OSSpinLock
```
#import <Foundation/Foundation.h>
#import <objc/runtime.h>
#import <objc/message.h>
#import <libkern/OSAtomic.h>
#import <pthread.h>

#define ITERATIONS (1024*1024*32)

static unsigned long long disp=0, land=0;

int main(){
    double then, now;
    unsigned int i, count;
    pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
    OSSpinLock spinlock = OS_SPINLOCK_INIT;

    NSAutoreleasePool *pool = [NSAutoreleasePool new];

    NSLock *lock = [NSLock new];
    then = CFAbsoluteTimeGetCurrent();
    for(i=0; i<ITERATIONS; ++i)
    {
        [lock lock];
        [lock unlock];
    }
    now = CFAbsoluteTimeGetCurrent();
    printf("NSLock: %f sec\n", now-then);    

    then = CFAbsoluteTimeGetCurrent();
    IMP lockLock = [lock methodForSelector:@selector(lock)];
    IMP unlockLock = [lock methodForSelector:@selector(unlock)];
    
    for(i=0; i<ITERATIONS; ++i) {
        lockLock(lock,@selector(lock));
        unlockLock(lock,@selector(unlock));
    }
    now = CFAbsoluteTimeGetCurrent();
    printf("NSLock+IMP Cache: %f sec\n", now-then);    

    then = CFAbsoluteTimeGetCurrent();
    for(i=0;i<ITERATIONS;++i) {
        pthread_mutex_lock(&mutex);
        pthread_mutex_unlock(&mutex);
    }
    now = CFAbsoluteTimeGetCurrent();
    printf("pthread_mutex: %f sec\n", now-then);

    then = CFAbsoluteTimeGetCurrent();
    for(i=0; i<ITERATIONS; ++i) {
        OSSpinLockLock(&spinlock);
        OSSpinLockUnlock(&spinlock);
    }
    now = CFAbsoluteTimeGetCurrent();
    printf("OSSpinlock: %f sec\n", now-then);

    id obj = [NSObject new];

    then = CFAbsoluteTimeGetCurrent();
    for(i=0; i<ITERATIONS; ++i){
        @synchronized(obj) { }
    }
    now = CFAbsoluteTimeGetCurrent();
    printf("@synchronized: %f sec\n", now-then);

    [pool release];
    return 0;
}
```

```
NSLock: 3.5175 sec
NSLock+IMP Cache: 3.1165 sec
Mutex: 1.5870 sec
SpinLock: 1.0893
@synchronized: 9.9488 sec
```

### 分析
1. @synchronized 非常沉重的api, 因为他需要额外设置一个exception handler, 并且结束的时候，需要少量内部锁。
2. OSSpinLock 不需要进入内核，只是简单的重复检测锁是否释放。如果此任务需要执行很久的话，效率会非常低。但是由于它节省了系统调用和上下文切换的时间，所以当任务确实很快的时候，效率很好。
3. Pthread mutexes 首先采用OSSpinLock来检测，如果没有结束，则进入内核锁方案
4. NSLock 是针对pthread mutexes的封装，因为多了Objc环境的调用所以效率较低。（have to pay for the extra ObjC overhead）

###原子操作
```
void DWDispatchOnce(dispatch_once_t *predicate, dispatch_block_t block) {
    if(*predicate == 2) {
        __sync_synchronize();
        return;
    }
 
    volatile dispatch_once_t *volatilePredicate = predicate;
 
    if(__sync_bool_compare_and_swap(volatilePredicate, 0, 1)) {
        block();
        __sync_synchronize();
        *volatilePredicate = 2;
    } else {
        while(*volatilePredicate != 2)
            ;//注意这里没有循环体
        __sync_synchronize();
    }
}
```
首先检查predicate是否为2，假如为2，则调用__sync_synchronize这个builtin函数并返回，调用此函数会产生一个memory barrier，用以保证cpu读写顺序严格按照程序的编写顺序来进行，关于memory barrier的更多信息，还是查wiki吧。
紧接着是一个volatile修饰符修饰的指针临时变量，如此编译器就会假定此指针指向的值可能会随时被其它线程改变，从而防止编译器对此指针指向的值的读写进行优化，比如cache，reorder等。
然后进行“原子比较交换”，如果predicate为0，则将predicate置为1，表示正在执行block，并返回true，如此便进入了block执行分支，在block执行完毕之后，我们依旧需要一个memory barrier，最后我们将predicate置为2，表示执行已经完成，后续调用应该直接返回。
当某个线程A正在执行block时，任何线程B再进入此函数，便会进入else分支，然后在此分支中进行等待，直至线程A将predicate置为2，然后调用__sync_synchronize并返回
这个实现是线程安全的，并且是无锁的，但是，依旧需要消耗11.5ns来执行，比自旋锁都慢，实际上memory barrier是很慢的。至于为什么比自旋锁还慢，memory barrier有好几种，__sync_synchronize产生的是mfenceCPU指令，是最蛋疼的一种，跟那蛋疼到忧伤的SSE4指令集是一路货。但不管怎么样，我想说的是，memory barrier是有不小的开销的

### dispatch_semaphore

```
dispatch_queue_t ioQueue = dispatch_queue_create("com.dreamingwish.imagegcd.io", NULL);
 
int cpuCount = [[NSProcessInfo processInfo] processorCount];
dispatch_semaphore_t jobSemaphore = dispatch_semaphore_create(cpuCount * 2);
 
dispatch_group_t group = dispatch_group_create();
__block uint32_t count = -1;
for(NSString *path in enumerator)
{
    WithAutoreleasePool(^{
        if([[[path pathExtension] lowercaseString] isEqual: @"jpg"])
        {
            NSString *fullPath = [dir stringByAppendingPathComponent: path];
             
            dispatch_semaphore_wait(jobSemaphore, DISPATCH_TIME_FOREVER);
         
            dispatch_group_async(group, ioQueue, BlockWithAutoreleasePool(^{
                NSData *data = [NSData dataWithContentsOfFile: fullPath];
                dispatch_group_async(group, globalQueue, BlockWithAutoreleasePool(^{
                    NSData *thumbnailData = ThumbnailDataForData(data);
                    if(thumbnailData) {
                        NSString *thumbnailName = [NSString stringWithFormat: @"%d.jpg",
                                                   OSAtomicIncrement32(&count;)];
                        NSString *thumbnailPath = [destination stringByAppendingPathComponent: thumbnailName];
                        dispatch_group_async(group, ioQueue, BlockWithAutoreleasePool(^{
                            [thumbnailData writeToFile: thumbnailPath atomically: NO];
                            dispatch_semaphore_signal(jobSemaphore);
                        }));
                    }
                    else{
                        dispatch_semaphore_signal(jobSemaphore);
                    };
                }));
            }));
        }
    });
}
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
```

dispatch_semaphore_wait() has only decremented the semaphore value if it returns 0


### dispatch_source
```
NSArray* indexPathsAry = @[ ... ];
const NSTimeInterval intervalInSeconds = 1.0 / 30.0;

__block NSUInteger i = 0;

dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, dispatch_get_main_queue());
dispatch_source_set_timer(timer, DISPATCH_TIME_NOW,  (uint64_t)(intervalInSeconds * NSEC_PER_SEC), 0);
dispatch_source_set_event_handler(timer, ^{
    NSIndexPath *indexPath = indexPathsAry[i++];
    id item = items[indexPath.row];
    [items removeObject:item];

    [tableView beginUpdates];
    [tableView deleteRowsAtIndexPaths:indexPath withAnimation:...];
    [tableView endUpdates];

    if (i >= indexPathsAry.count) {
        dispatch_source_cancel(timer);
    }
});

dispatch_source_set_cancel_handler(timer, ^{
    // whatever you want to happen when all the removes are done.
})
dispatch_resume(timer);

```

```
NSOperation* finishOp = [NSBlockOperation blockOperationWithBlock:^{
    // ... whatever you want to have happen when all deletes have been processed.
}];

for (NSIndexPath* toDelete in indexPathsAry)
{
    NSOperation* op = [NSBlockOperation blockOperationWithBlock:^{
        id item = items[toDelete.row];
        [items removeObject: item];

        [tableView beginUpdates];
        [tableView deleteRowsAtIndexPaths: @[toDelete] withAnimation:...];
        [tableView endUpdates];
    }];
    [finishOp addDependency: op];
    [[NSOperationQueue mainQueue] addOperation: op];
}
[[NSOperationQueue mainQueue] addOperation: finishOp];
```

