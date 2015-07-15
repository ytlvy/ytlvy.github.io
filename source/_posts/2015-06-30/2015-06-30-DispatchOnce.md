title: DispatchOnce
categories: IOS
tags:
  - IOS
  - Dispatch Once
description:
date: 2015-06-30 15:31:31
author:
photos:
---

![](/images/DispatchOnce.jpg)

## Dispatch_Once 

四种场景如上图
1. 第一次执行，`block调用`，调用结束后需要置标记变量
2. 非第一次执行，而此时#1`block调用`尚未完成(预执行拿到的 obj 为 nil, 此时需要废弃此次预执行)，线程需进入等待#1状态
3. 非第一次执行，而此时#1`block调用`已经完成(预执行拿到的 obj 已经初始化完毕)此时线程的预执行是正确的, 之后可能继续执行,也可能由于时间问题,废弃此次预执行进入等待
4. 非第一次执行，而此时#1已经完成，线程预执行后, 直接跳过block而进行后续任务

Dispatch_Once 保证了当第一个线程在进行alloc 对象时，有其他线程发起, 会废弃此线程的预执行，进入等待。在第一个线程结束时，将正在等待的线程唤醒。而实现这一机制的方法是在生成对象和变更identifier之间加入一个大于预执行时间的等待来实现的。

### Dispatch_Once实现用到的技术
- 原子操作 "原子比较交换函数" __sync_bool_compare_and_swap
- cpuid指令 实现大于预执行时间的延迟等待
- dispatch_thread_semaphore 来实现线程之间的等待和唤醒
<!-- more -->
### cpu的分支预测和预执行
> &nbsp;&nbsp;&nbsp;&nbsp;流水线特性使得CPU能更快地执行线性指令序列，但是当遇到条件判断分支时，麻烦来了，在判定语句返回结果之前，cpu不知道该执行哪个分支，那就得等着（术语叫做pipeline stall），这怎么能行呢，所以，CPU会进行预执行，cpu先猜测一个可能的分支，然后开始执行分支中的指令。现代CPU一般都能做到超过90%的猜测命中率，这可比NBA选手发球命中率高多了。然后当判定语句返回，加入cpu猜错分支，那么之前进行的执行都会被抛弃，然后从正确的分支重新开始执行。
&nbsp;&nbsp;&nbsp;&nbsp;在dispatch_once中，唯一一个判断分支就是predicate，dispatch_once会让CPU预执行条件不成立的分支，这样可以大大提升函数执行速度。但是这样的预执行导致的结果是使用了未初始化的obj并将函数返回，这显然不是预期结果。

### 不对称barrier
> &nbsp;&nbsp;&nbsp;&nbsp;在intel的CPU上，dispatch_once动用了cpuid指令来达成这个目的。cpuid本来是用作取得cpu的信息，但是这个指令也同时强制将指令流串行化，并且这个指令是需要比较长的执行时间的(在某些cpu上，甚至需要几百圈cpu时钟).这个指令保证了, 当其他线程预执行读取到 nil时,可以废弃当前的预执行操作,从而进入 dispatch_once 的 else 方法进入等待.

### dispatch_once读取端的实现
> DISPATCH_EXPECT 是用来告诉cpu *predicate等于~0l是更有可能的判定结果，这就使得cpu能猜测到更正确的分支，并提高效率


```
DISPATCH_INLINE DISPATCH_ALWAYS_INLINE DISPATCH_NONNULL_ALL DISPATCH_NOTHROW
void
_dispatch_once(dispatch_once_t *predicate, dispatch_block_t block)
{
    if (DISPATCH_EXPECT(*predicate, ~0l) != ~0l) {
        dispatch_once(predicate, block);
    }
}
#define dispatch_once _dispatch_once
```

### dispatch_once的实现

```
struct _dispatch_once_waiter_s {
    volatile struct _dispatch_once_waiter_s *volatile dow_next;
    _dispatch_thread_semaphore_t dow_sema;
};
 
#define DISPATCH_ONCE_DONE ((struct _dispatch_once_waiter_s *)~0l)
 
#ifdef __BLOCKS__
void
dispatch_once(dispatch_once_t *val, dispatch_block_t block)
{
    struct Block_basic *bb = (void *)block;
 
    dispatch_once_f(val, block, (void *)bb->Block_invoke);
}
#endif
 
DISPATCH_NOINLINE
void
dispatch_once_f(dispatch_once_t *val, void *ctxt, dispatch_function_t func)
{
    struct _dispatch_once_waiter_s * volatile *vval = (struct _dispatch_once_waiter_s**)val;
    struct _dispatch_once_waiter_s dow = { NULL, 0 };
    struct _dispatch_once_waiter_s *tail, *tmp;
    _dispatch_thread_semaphore_t sema;
 
    if (dispatch_atomic_cmpxchg(vval, NULL, &dow)) {
        dispatch_atomic_acquire_barrier();//这是一个空的宏函数，什么也不做
        _dispatch_client_callout(ctxt, func);
        dispatch_atomic_maximally_synchronizing_barrier();
        //dispatch_atomic_release_barrier(); // assumed contained in above
        tmp = dispatch_atomic_xchg(vval, DISPATCH_ONCE_DONE);
        tail = &dow;
        while (tail != tmp) {
            while (!tmp->dow_next) {
                _dispatch_hardware_pause();
            }
            sema = tmp->dow_sema;
            tmp = (struct _dispatch_once_waiter_s*)tmp->dow_next;
            _dispatch_thread_semaphore_signal(sema);
        }
    } else {
        dow.dow_sema = _dispatch_get_thread_semaphore();
        for (;;) {
            tmp = *vval;
            if (tmp == DISPATCH_ONCE_DONE) {
                break;
            }
            dispatch_atomic_store_barrier();
            if (dispatch_atomic_cmpxchg(vval, tmp, &dow)) {
                dow.dow_next = tmp;
                _dispatch_thread_semaphore_wait(dow.dow_sema);
            }
        }
        _dispatch_put_thread_semaphore(dow.dow_sema);
    }
}
```

> dispatch_atomic_cmpxchg 是 “原子比较交换函数”__sync_bool_compare_and_swap的宏替换

####  执行block的分支
&nbsp;&nbsp;&nbsp;&nbsp;当dispatch_once第一次执行时，predicate也即val为0，那么此“原子比较交换函数”将返回true并将vval指向值赋值为&dow，即为“等待中”，_dispatch_client_callout其内部做了一些判定，但实际上是调用了func而已。到此，block中的用户代码执行完毕。
&nbsp;&nbsp;&nbsp;&nbsp; 接下来就是上篇提及的cpuid指令等待，使得其他线程的【读取到未初始化值的】预执行能被判定为猜测未命中，从而使得这些线程能够进入dispatch_once_f里的另一个分支从而进行等待。
&nbsp;&nbsp;&nbsp;&nbsp; cpuid指令完毕后，调用dispatch_atomic_xchg进行赋值，置其为DISPATCH_ONCE_DONE，即“完成”，这里dispatch_atomic_xchg是内建“原子交换函数”__sync_swap的优化版宏替换，其将第二个参数的值赋给第一个参数（解引用指针），然后返回第一个参数被赋值前的解引用值，其原型为：
> type __sync_swap(type *ptr, type value, ...)

接下来是对信号量链的处理：
1. 在block执行过程中，没有其他线程进入本函数来等待，则vval指向值保持为&dow，即tmp被赋值为&dow，即下方while循环不会被执行，此分支结束。
2. 在block执行过程中，有其他线程进入本函数来等待，那么会构造一个信号量链表（vval指向值变为信号量链的头部，链表的尾部为&dow），此时就会进入while循环，在此while循环中，遍历链表，逐个signal每个信号量，然后结束循环。

> while (!tmp->dow_next)此循环是等待在&dow上，因为线程等待分支#2会中途将val赋值为&dow，然后为->dow_next赋值，这期间->dow_next值为NULL，需要等待，详见下面线程等待分支#2的描述
> _dispatch_hardware_pause此句是为了提示cpu减少额外处理，提升性能，节省电力

#### 线程等待分支
当执行block分支#1未完成，且有线程再进入本函数时，将进入线程等待分支：
1. 先调用_dispatch_get_thread_semaphore创建一个信号量，此信号量被赋值给dow.dow_sema
2. 然后进入一个无限for循环，假如发现vval的指向值已经为DISPATCH_ONCE_DONE，即“完成”，则直接break，然后调用_dispatch_put_thread_semaphore函数销毁信号量并退出函数

> _dispatch_get_thread_semaphore内部使用的是“有即取用，无即创建”策略来获取信号量。
> _dispatch_put_thread_semaphore内部使用的是“销毁旧的，存储新的”策略来缓存信号量。

&nbsp;&nbsp;&nbsp;&nbsp; 假如vval的解引用值并非DISPATCH_ONCE_DONE，则进行一个“原子比较并交换”操作（此操作可以避免两个等待线程同时操作链表带来的问题），假如此时vval指向值已不再是tmp（这种情况发生在多个线程同时进入线程等待分支#2，并交错修改链表）则for循环重新开始，再尝试重新获取一次vval来进行同样的操作；若指向值还是tmp，则将vval的指向值赋值为&dow，此时val->dow_next值为NULL，可能会使得block执行分支#1进行while等待（如前述），紧接着执行dow.dow_next = tmp这句来增加链表节点（同时也使得block执行分支#1的while等待结束），然后等待在信号量上，当block执行分支#1完成并遍历链表来signal时，唤醒、释放信号量，然后一切就完成了。

### 小结
1. 线程A执行Block时，任何其它线程都需要等待。
2. 线程A执行完Block应该立即标记任务完成状态，然后遍历信号量链来唤醒所有等待线程。
3. 线程A遍历信号量链来signal时，任何其他新进入函数的线程都应该直接返回而无需等待。
4. 线程A遍历信号量链来signal时，若有其它等待线程B仍在更新或试图更新信号量链，应该保证此线程B能正确_完成其任务：a.直接返回 b.等待在信号量上并很快又被唤醒。
5. 线程B构造信号量时，应该考虑线程A随时可能改变状态（“等待”、“完成”、“遍历信号量链”）。
6. 线程B构造信号量时，应该考虑到另一个线程C也可能正在更新或试图更新信号量链，应该保证B、C都能正常_完成其任务：a.增加链节并等待在信号量上 b.发现线程A已经标记“完成”然后直接销毁信号量并退出函数。


