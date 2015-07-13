title: CFRunLoop examples
categories: IOS
tags:
  - IOS
  - CFRunLoop
description:
date: 2015-06-30 09:16:56
author:
photos:
---
### 第一个

```
#include <CoreFoundation/CoreFoundation.h>  
  
static void  
_perform(void *info __unused)  
{  
    printf("hello\n");  
}  
  
static void  
_timer(CFRunLoopTimerRef timer __unused, void *info)  
{  
    CFRunLoopSourceSignal(info);  
}  
  
int  
main()  
{  
    CFRunLoopSourceRef source;  
    CFRunLoopSourceContext source_context;  
    CFRunLoopTimerRef timer;  
    CFRunLoopTimerContext timer_context;  
  
    bzero(&source_context, sizeof(source_context));  
    source_context.perform = _perform;  
    source = CFRunLoopSourceCreate(NULL, 0, &source_context);  
    CFRunLoopAddSource(CFRunLoopGetCurrent(), source, kCFRunLoopCommonModes);  
  
    bzero(&timer_context, sizeof(timer_context));  
    timer_context.info = source;  
    timer = CFRunLoopTimerCreate(NULL, CFAbsoluteTimeGetCurrent(), 1, 0, 0, 
    _timer, &timer_context);  
    CFRunLoopAddTimer(CFRunLoopGetCurrent(), timer, kCFRunLoopCommonModes);  
  
    CFRunLoopRun();  
  
    return 0;  
}  
```

<!-- more -->

### 第二个：
```
#include <dispatch/dispatch.h>  
#include <stdio.h>  
  
int  
main()  
{  
    dispatch_source_t source, timer;  
  
    source = dispatch_source_create(DISPATCH_SOURCE_TYPE_DATA_ADD, 0, 0,
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0));  
    dispatch_source_set_event_handler(source, ^{  
        printf("hello\n");  
    });  
    dispatch_resume(source);  
  
    timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, 
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0));  
    dispatch_source_set_timer(timer, DISPATCH_TIME_NOW, 1ull * NSEC_PER_SEC, 0);  
    dispatch_source_set_event_handler(timer, ^{  
        dispatch_source_merge_data(source, 1);  
    });  
    dispatch_resume(timer);  
  
    dispatch_main();  
}  
```

功能是向main线程中加入两个input source，一个是timer，一个是自定义input source，然后这个timer中触发自定义source，于是调用其回调方法。 在这儿timer触发source来调用回调方法，显得有点多此一举。但是在多线程开发当中，这就很有用了，我们可以把这个自定义的source加入到子线程的runloop中，然后在主线程中触发source，这样在子线程中就可以调用回调方法了。  这样做的好久是什么呀？ 节约用电，因为runloop一般情况下是休眠的，只有事件触发的时候才开始工作。 这与windows下的waitforsingleobject有点类似， 与多线程中的信号量，事件也有些雷同。
 
上面说到的input source（输入源例）到底是什么呢？输入源样例可能包括用户输入设备（如点击button）、网络链接(socket收到数据)、定期或时间延迟事件（NSTimer），还有异步回调(NSURLConnection的异步请求)。然后我们对其进行了分类，有三类可以被runloop监控，分别是sources、timers、observers。
在苹果文档中对runloop有详细介绍，下面参考中有中文版。那文档中的代码关于NSPort的部份在iOS上是不行的，不过可以用其CF方法实现，在我的demo中有展示。
 
每一个线程都有自己的runloop, 主线程是默认开启的，创建的子线程要手动开启，因为NSApplication 只启动main applicaiton thread。
没有source的runloop会自动结束。
事件由NSRunLoop 类处理。
RunLoop监视操作系统的输入源，如果没有事件数据， 不消耗任何CPU 资源。
如果有事件数据，run loop 就发送消息，通知各个对象。
用 currentRunLoop 获得 runloop的 reference
给 runloop 发送run 消息启动它。
 

文档中介绍下面四种情况是使用runloop的场合：
 1.使用端口或自定义输入源和其他线程通信
 2.子线程中使用了定时器
 3.cocoa中使用任何performSelector到了线程中运行方法
 4.使线程履行周期性任务，（我把这个理解与2相同）
如果我们在子线程中用了NSURLConnection异步请求，那也需要用到runloop，不然线程退出了，相应的delegate方法就不能触发。

### 用户事件事例：累计更新
```
dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_DATA_ADD, 0, 0, dispatch_get_main_queue());
dispatch_source_set_event_handler(source, ^{
    [progressIndicator incrementBy:dispatch_source_get_data(source)];
});
dispatch_resume(source);
 
dispatch_apply([array count], globalQueue, ^(size_t index) {
    // do some work on data at index
    dispatch_source_merge_data(source, 1);
});
```
假设你已经将进度条的min/max值设置好了，那么这段代码就完美了。数据会被并发处理。当每一段数据完成后，会通知dispatch source并将dispatch source data加1，这样我们就认为一个单元的工作完成了。事件句柄根据已完成的工作单元来更新进度条。若主线程比较空闲并且这些工作单元进行的比较慢，那么事件句柄会在每个工作单元完成的时候被调用，实时更新。如果主线程忙于其他工作，或者工作单元完成速度很快，那么完成事件会被联结起来，导致进度条只在主线程变得可用时才被更新，并且一次将积累的改变更新至GUI。使用的dispatch source而不使用dispatch_async的唯一原因就是利用联结的优势。

### 内建事件
```
dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_source_t stdinSource = dispatch_source_create(DISPATCH_SOURCE_TYPE_READ,
                                                       STDIN_FILENO,
                                                       0,
                                                       globalQueue);
dispatch_source_set_event_handler(stdinSource, ^{
    char buf[1024];
    int len = read(STDIN_FILENO, buf, sizeof(buf));
    if(len > 0)
        NSLog(@"Got data from stdin: %.*s", len, buf);
});
dispatch_resume(stdinSource);
```

这是标准的UNIX方式来处理事务的好处，不用去写loop。如果使用经典的 read调用，我们还得万分留神，因为返回的数据可能比请求的少，还得忍受无厘头的“errors”，比如 EINTR (系统调用中断)。使用GCD，我们啥都不用管，就从这些蛋疼的情况里解脱了。如果我们在文件描述符中留下了未读取的数据，GCD会再次调用我们的句柄。
对于标准输入，这没什么问题，但是对于其他文件描述符，我们必须考虑在完成读写之后怎样清除描述符。对于dispatch source还处于活跃状态时，我们决不能关闭描述符。如果另一个文件描述符被创建了（可能是另一个线程创建的）并且新的描述符刚好被分配了相同的数字，那么你的dispatch source可能会在不应该的时候突然进入读写状态。de这个bug可不是什么好玩的事儿。
适当的清除方式是使用 dispatch_source_set_cancel_handler，并传入一个block来关闭文件描述符。然后我们使用 dispatch_source_cancel来取消dispatch source，使得句柄被调用，然后文件描述符被关闭。
使用其他dispatch source类型也差不多。总的来说，你提供一个source（mach port、文件描述符、进程ID等等）的区分符来作为diapatch source的句柄。mask参数通常不会被使用，但是对于 DISPATCH_SOURCE_TYPE_PROC 来说mask指的是我们想要接受哪一种进程事件。然后我们提供一个句柄，然后恢复这个source（前面我加粗字体所说的，得先恢复），搞定。dispatch source也提供一个特定于source的data，我们使用 dispatch_source_get_data函数来访问它。例如，文件描述符会给出大致可用的字节数。进程source会给出上次调用之后发生的事件的mask。具体每种source给出的data的含义，