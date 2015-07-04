title: Dispatch_semaphore
categories: IOS
tags:
  - IOS
  - dispatch semaphore
description: dispatch_semaphore 介绍以及使用
date: 2015-06-30 16:28:07
author:
photos:
---
![](/images/DispatchSemaphore.jpg)

## Dispatch_semaphore
`semaphore` 通过`dispatch_semaphore_create`创建一个信号量来实现多线程并发数控制, 此信号量为最高可进行的并发数. 通过`dispatch_semaphore_wait`来消耗一个信号量,并进入其后的执行逻辑, 在逻辑执行完毕后,通过`dispatch_semaphore_signal`来释放一个信号量, 并唤醒正在等待的线程. 如果`dispatch_semaphore_wait`操作时,信号量为0则进入等待状态.

### functions
- dispatch_semaphore_create
- dispatch_semaphore_wait (减少一个信号量)
- dispatch_semaphore_signal (增加一个信号量)

### 实例

```
- (void)test4{
    int data = 3;
    __block int mainData = 0;
    __block int sum = 0;
    __block dispatch_semaphore_t sem = dispatch_semaphore_create(2);
    dispatch_queue_t ioQueue = dispatch_queue_create("com.dreamingwish.imagegcd.io", DISPATCH_QUEUE_SERIAL);
    
    dispatch_queue_t queue = dispatch_queue_create("StudyBlocks", DISPATCH_QUEUE_CONCURRENT);
    dispatch_group_t group = dispatch_group_create();
    
    for (int i =0 ; i<100; i++) {
        @autoreleasepool {
            dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);  // -1 非零即通过
            dispatch_group_async(group, ioQueue, ^{
//                dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);
                dispatch_group_async(group, queue, ^{
                    sum += data;
                    
                    __block int nsum = sum;
                    dispatch_group_async(group, ioQueue, ^{
                        NSLog(@" >> Sum: %d", nsum);
                        
                        dispatch_semaphore_signal(sem);  // +1
                    });
                    
                });
            });
        }
    }
    dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
//    dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);
    for (int j =0; j<5; j++) {
        mainData++;
        NSLog(@" >> MainData: %d", mainData);
    }
//    dispatch_semaphore_signal(sem);
}
```


```
- (void)test3{
    int data = 3;
    __block int mainData = 0;
    __block dispatch_semaphore_t sem = dispatch_semaphore_create(1);
    
    dispatch_queue_t queue = dispatch_queue_create("StudyBlocks", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);
    
    dispatch_async(queue, ^{
        int sum = 0;
        
        for (int i=0; i < 5; i++) {
            sum += data;
            NSLog(@" >> Sum: %d", sum);
        }
        
        dispatch_semaphore_signal(sem);
    });
    
    
    for (int j =0; j<5; j++) {
        mainData++;
        NSLog(@" >> MainData: %d", mainData);
    }

}
```



