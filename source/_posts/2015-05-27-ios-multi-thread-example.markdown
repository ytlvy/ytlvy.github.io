---
layout: post
title: "ios 多线程操作"
date: 2015-05-27 15:50:39 +0800
comments: true
categories: IOS
---
### 多线程操作
1. 创建一个NSOperation不应该直接调用start方法（如果直接start则会在主线程中调用）而是应该放到NSOperationQueue中启动
2. NSOperation进行多线程开发可以控制线程总数及线程依赖关系
3. 相比NSInvocationOperation推荐使用NSBlockOperation，代码简单，同时由于闭包性使它没有传参问题。
4. NSOperation是对GCD面向对象的ObjC封装，但是相比GCD基于C语言开发，效率却更高，建议如果任务之间有依赖关系或者想要监听任务完成状态的情况下优先选择NSOperation否则使用GCD
5. 在GCD中串行队列中的任务被安排到一个单一线程执行（不是主线程），可以方便地控制执行顺序；并发队列在多个线程中执行（前提是使用异步方法），顺序控制相对复杂，但是更高效
6. 在GDC中一个操作是多线程执行还是单线程执行取决于当前队列类型和执行方法，只有队列类型为并行队列并且使用异步方法执行时才能在多个线程中执行（如果是并行队列使用同步方法调用则会在主线程中执行）
7. 相比使用NSLock，@synchronized更加简单，推荐使用后者

<!--more-->

#### NSThread
不能控制执行顺序 参数一个

    -(void)loadImageWithMultiThread{
        //方法1：使用对象方法
        //创建一个线程，第一个参数是请求的操作，第二个参数是操作方法的参数
        //    NSThread *thread=[[NSThread alloc]initWithTarget:self selector:@selector(loadImage) object:nil];
        //    //启动一个线程，注意启动一个线程并非就一定立即执行，而是处于就绪状态，当系统调度时才真正执行
        //    [thread start];
        
        //方法2：使用类方法
        [NSThread detachNewThreadSelector:@selector(loadImage) toTarget:self withObject:nil];
    }


    -(void)loadImageWithMultiThread{
        //创建多个线程用于填充图片
        for (int i=0; i<ROW_COUNT*COLUMN_COUNT; ++i) {
            // [NSThread detachNewThreadSelector:@selector(loadImage:) toTarget:self withObject:[NSNumber numberWithInt:i]];
            NSThread *thread=[[NSThread alloc]initWithTarget:self selector:@selector(loadImage:) object:[NSNumber numberWithInt:i]];
            thread.name=[NSString stringWithFormat:@"myThread%i",i];//设置线程名称
            [thread start];
        }
    }


    -(void)loadImageWithMultiThread{
        NSMutableArray *threads=[NSMutableArray array];
        int count=ROW_COUNT*COLUMN_COUNT;
        //创建多个线程用于填充图片
        for (int i=0; i<count; ++i) {
    //        [NSThread detachNewThreadSelector:@selector(loadImage:) toTarget:self withObject:[NSNumber numberWithInt:i]];
            NSThread *thread=[[NSThread alloc]initWithTarget:self selector:@selector(loadImage:) object:[NSNumber numberWithInt:i]];
            thread.name=[NSString stringWithFormat:@"myThread%i",i];//设置线程名称
            if(i==(count-1)){
                thread.threadPriority=1.0;
            }else{
                thread.threadPriority=0.0;
            }
            [threads addObject:thread];
        }
        
        for (int i=0; i<count; i++) {
            NSThread *thread=threads[i];
            [thread start];
        }
    }


#### NSThread stop
    #pragma mark 停止加载图片
    -(void)stopLoadImage{
        for (int i=0; i<ROW_COUNT*COLUMN_COUNT; i++) {
            NSThread *thread= _threads[i];
            //判断线程是否完成，如果没有完成则设置为取消状态
            //注意设置为取消状态仅仅是改变了线程状态而言，并不能终止线程
            if (!thread.isFinished) {
                [thread cancel];
                
            }
        }
    }

    #pragma mark 加载图片
    -(void)loadImage:(NSNumber *)index{
        int i=[index integerValue];

        NSData *data= [self requestData:i];

        
        NSThread *currentThread=[NSThread currentThread];
        
    //    如果当前线程处于取消状态，则退出当前线程
        if (currentThread.isCancelled) {
            NSLog(@"thread(%@) will be cancelled!",currentThread);
            [NSThread exit];//取消当前线程
        }
        
        KCImageData *imageData=[[KCImageData alloc]init];
        imageData.index=i;
        imageData.data=data;
        [self performSelectorOnMainThread:@selector(updateImage:) withObject:imageData waitUntilDone:YES];
    }

    #pragma mark 多线程下载图片
    -(void)loadImageWithMultiThread{
        int count=ROW_COUNT*COLUMN_COUNT;
        _threads=[NSMutableArray arrayWithCapacity:count];
        
        //创建多个线程用于填充图片
        for (int i=0; i<count; ++i) {
            NSThread *thread=[[NSThread alloc]initWithTarget:self selector:@selector(loadImage:) object:[NSNumber numberWithInt:i]];
            thread.name=[NSString stringWithFormat:@"myThread%i",i];//设置线程名称
            [_threads addObject:thread];
        }
        //循环启动线程
        for (int i=0; i<count; ++i) {
            NSThread *thread= _threads[i];
            [thread start];
        }
    }

#### NSInvocationOperation

    #pragma mark 请求图片数据
    -(NSData *)requestData:(int )index{
        //对于多线程操作建议把线程操作放到@autoreleasepool中
        @autoreleasepool {
            NSURL *url=[NSURL URLWithString:_imageNames[index]];
            NSData *data=[NSData dataWithContentsOfURL:url];

            return data;
        }
    }

    -(void)loadImageWithMultiThread{
        /*创建一个调用操作
         object:调用方法参数
        */
        NSInvocationOperation *invocationOperation=[[NSInvocationOperation alloc]initWithTarget:self selector:@selector(loadImage) object:nil];
        //创建完NSInvocationOperation对象并不会调用，它由一个start方法启动操作，但是注意如果直接调用start方法，则此操作会在主线程中调用，一般不会这么操作,而是添加到NSOperationQueue中
    //    [invocationOperation start];
        
        //创建操作队列
        NSOperationQueue *operationQueue=[[NSOperationQueue alloc]init];
        //注意添加到操作队后，队列会开启一个线程执行此操作
        [operationQueue addOperation:invocationOperation];
    }

#### NSBlockOperation
    #pragma mark 多线程下载图片
    -(void)loadImageWithMultiThread{
        int count=ROW_COUNT*COLUMN_COUNT;
        //创建操作队列
        NSOperationQueue *operationQueue=[[NSOperationQueue alloc]init];
        operationQueue.maxConcurrentOperationCount=5;//设置最大并发线程数
        //创建多个线程用于填充图片
        for (int i=0; i<count; ++i) {
            //方法1：创建操作块添加到队列
    //        //创建多线程操作
    //        NSBlockOperation *blockOperation=[NSBlockOperation blockOperationWithBlock:^{
    //            [self loadImage:[NSNumber numberWithInt:i]];
    //        }];
    //        //创建操作队列
    //
    //        [operationQueue addOperation:blockOperation];
            
            //方法2：直接使用操队列添加操作
            [operationQueue addOperationWithBlock:^{
                [self loadImage:[NSNumber numberWithInt:i]];
            }];
            
        }
    }
    @end

#### 线程执行顺序
    -(void)loadImageWithMultiThread{
        int count=ROW_COUNT*COLUMN_COUNT;
        //创建操作队列
        NSOperationQueue *operationQueue=[[NSOperationQueue alloc]init];
        operationQueue.maxConcurrentOperationCount=5;//设置最大并发线程数
        
        NSBlockOperation *lastBlockOperation=[NSBlockOperation blockOperationWithBlock:^{
            [self loadImage:[NSNumber numberWithInt:(count-1)]];
        }];
        //创建多个线程用于填充图片
        for (int i=0; i<count-1; ++i) {
            //方法1：创建操作块添加到队列
            //创建多线程操作
            NSBlockOperation *blockOperation=[NSBlockOperation blockOperationWithBlock:^{
                [self loadImage:[NSNumber numberWithInt:i]];
            }];
            //设置依赖操作为最后一张图片加载操作
            [blockOperation addDependency:lastBlockOperation];
            
            [operationQueue addOperation:blockOperation];
            
        }
        //将最后一个图片的加载操作加入线程队列
        [operationQueue addOperation:lastBlockOperation];
    }

#### GCD
GCD中多线程操作方法不需要使用@autoreleasepool，GCD会管理内存

    #pragma mark 多线程下载图片
    -(void)loadImageWithMultiThread{
        int count=ROW_COUNT*COLUMN_COUNT;
        
        /*创建一个串行队列
         第一个参数：队列名称
         第二个参数：队列类型
        */
        dispatch_queue_t serialQueue=dispatch_queue_create("myThreadQueue1", DISPATCH_QUEUE_SERIAL);//注意queue对象不是指针类型
        //创建多个线程用于填充图片
        for (int i=0; i<count; ++i) {
            //异步执行队列任务
            dispatch_async(serialQueue, ^{
                [self loadImage:[NSNumber numberWithInt:i]];
            });
            
        }
        //非ARC环境请释放
    //    dispatch_release(seriQueue);
    }


#### 并发队列
    -(void)loadImageWithMultiThread{
        int count=ROW_COUNT*COLUMN_COUNT;
        
        /*取得全局队列
         第一个参数：线程优先级
         第二个参数：标记参数，目前没有用，一般传入0
        */
        dispatch_queue_t globalQueue=dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
        //创建多个线程用于填充图片
        for (int i=0; i<count; ++i) {
            //异步执行队列任务
            dispatch_async(globalQueue, ^{
                [self loadImage:[NSNumber numberWithInt:i]];
            });
        }
    }

#### NSLock
    -(NSData *)requestData:(int )index{
        NSData *data;
        NSString *name;
        //加锁
        [_lock lock];
        if (_imageNames.count>0) {
            name=[_imageNames lastObject];
            [_imageNames removeObject:name];
        }
        //使用完解锁
        [_lock unlock];
        if(name){
            NSURL *url=[NSURL URLWithString:name];
            data=[NSData dataWithContentsOfURL:url];
        }
        return data;
    }


#### @synchronized代码块
    -(NSData *)requestData:(int )index{
        NSData *data;
        NSString *name;
        //线程同步
        @synchronized(self){
            if (_imageNames.count>0) {
                name=[_imageNames lastObject];
                [NSThread sleepForTimeInterval:0.001f];
                [_imageNames removeObject:name];
            }
        }
        if(name){
            NSURL *url=[NSURL URLWithString:name];
            data=[NSData dataWithContentsOfURL:url];
        }
        return data;
    }

#### 使用GCD解决资源抢占问题
    dispatch_semaphore_t _semaphore;//定义一个信号量
    semaphore=dispatch_semaphore_create(1);

    -(NSData *)requestData:(int )index{
        NSData *data;
        NSString *name;
        
        /*信号等待
         第二个参数：等待时间
         */
        dispatch_semaphore_wait(_semaphore, DISPATCH_TIME_FOREVER);
        if (_imageNames.count>0) {
            name=[_imageNames lastObject];
            [_imageNames removeObject:name];
        }
        //信号通知
        dispatch_semaphore_signal(_semaphore);

        
        if(name){
            NSURL *url=[NSURL URLWithString:name];
            data=[NSData dataWithContentsOfURL:url];
        }
        
        return data;
    }

#### 扩展--控制线程通信
由于线程的调度是透明的，程序有时候很难对它进行有效的控制，为了解决这个问题iOS提供了NSCondition来控制线程通信(同前面GCD的信号机制类似)NSCondition实现了NSLocking协议，所以它本身也有lock和unlock方法，因此也可以将它作为NSLock解决线程同步问题    
