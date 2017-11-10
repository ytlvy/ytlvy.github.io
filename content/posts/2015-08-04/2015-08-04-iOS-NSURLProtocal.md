---
author: null
categories:
- IOS
date: '2015-08-04'
description: null
photos: null
tags:
- Network
title: iOS NSURLProtocal
---

[转自](http://xiangwangfeng.com/2014/11/29/NSURLProtocol%E5%92%8CNSRunLoop%E7%9A%84%E9%82%A3%E4%BA%9B%E5%9D%91/)
最近用AFNetworking替换掉了工程里的ASIHttpRequest，结果陆续碰到很多问题：

- 如何统一地添加全局的HTTP头(不仅仅是UA而已)
- 如何优雅地进行流量统计
- 对特定的地址进行CDN加速(URL到IP的替换)
- 怎么实现HTTP的同步请求

前三个需求对于ASIHttpReqeust来说都不是问题，只需要在几个统一的点进行修改即可。而使用AFNetworking后就没有那么容易了：一方面AFNetworking中生成NSURLRequest的点比较多，并没有一个统一的路径。其次工程中会有部分直接使用NSURLConnecion的场景，无法统一。经cyzju提醒发现了NSURLProtocol这个大杀器，可惜对应的文档过于简略，唯一比较详细的介绍就只有RW的这篇教程而已，掉了很多坑，值得记上一笔。

## NSURLProtocol

### 概念
NSURLProtocol也是苹果众多黑魔法中的一种，使用它可以轻松地重定义整个URL Loading System。当你注册自定义NSURLProtocol后，就有机会对所有的请求进行统一的处理，基于这一点它可以让你
- 自定义请求和响应
- 提供自定义的全局缓存支持
- 重定向网络请求
- 提供HTTP Mocking (方便前期测试)
- 其他一些全局的网络请求修改需求

<!--more-->

### 使用方法
继承NSURLPorotocl，并注册你的NSURLProtocol
```
[NSURLProtocol registerClass:[YXURLProtocol class]];
```

当NSURLConnection准备发起请求时，它会遍历所有已注册的NSURLProtocol，询问它们能否处理当前请求。所以你需要尽早注册这个Protocol。
实现NSURLProtocol的相关方法
当遍历到我们自定义的NSURLProtocol时，系统先会调用canInitWithRequest:这个方法。顾名思义，这是整个流程的入口，只有这个方法返回YES我们才能够继续后续的处理。我们可以在这个方法的实现里面进行请求的过滤，筛选出需要进行处理的请求。
```
+ (BOOL)canInitWithRequest:(NSURLRequest *)request
{
    if ([NSURLProtocol propertyForKey:YXURLProtocolHandled inRequest:request])
    {
        return NO;
    }
    NSString *scheme = [[request URL] scheme];
    NSDictionary *dict = [request allHTTPHeaderFields];
    return [dict objectForKey:@"custom_header"] == nil &&
    ([scheme caseInsensitiveCompare:@"http"] == NSOrderedSame ||
     [scheme caseInsensitiveCompare:@"https"] == NSOrderedSame);
}
```

当筛选出需要处理的请求后，就可以进行后续的处理，需要至少实现如下4个方法
```
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request
{
    return request;
}
+ (BOOL)requestIsCacheEquivalent:(NSURLRequest *)a
                       toRequest:(NSURLRequest *)b
{
    return [super requestIsCacheEquivalent:a toRequest:b];
}
- (void)startLoading
{
    NSMutableURLRequest *mutableReqeust = [[self request] mutableCopy];
    [YXURLProtocol applyCustomHeaders:mutableReqeust];
    [NSURLProtocol setProperty:@(YES)
                        forKey:YXURLProtocolHandled
                     inRequest:mutableReqeust];
     
    self.connection = [NSURLConnection connectionWithRequest:mutableReqeust
                                                    delegate:self];
}
- (void)stopLoading
{
    [self.connection cancel];
    self.connection = nil;
}
```

- `canonicalRequestForRequest:` 返回规范化后的request,一般就只是返回当前request即可。
- `requestIsCacheEquivalent:toRequest: `用于判断你的自定义reqeust是否相同，这里返回默认实现即可。它的主要应用场景是某些直接使用缓存而非再次请求网络的地方。
- `startLoading` 和 `stopLoading` 实现请求和取消流程。


### 实现NSURLConnectionDelegate和NSURLConnectionDataDelegate
因为在第二步中我们接管了整个请求过程，所以需要实现相应的协议并使用NSURLProtocolClient将消息回传给URL Loading System。在我们的场景中推荐实现所有协议。
```
- (void)connection:(NSURLConnection *)connection
  didFailWithError:(NSError *)error
{
    [self.client URLProtocol:self
            didFailWithError:error];
}
- (NSURLRequest *)connection:(NSURLConnection *)connection willSendRequest:(NSURLRequest *)request redirectResponse:(NSURLResponse *)response
{
    if (response != nil) 
    {
        [[self client] URLProtocol:self wasRedirectedToRequest:request redirectResponse:response];
    }
    return request;
}
- (BOOL)connectionShouldUseCredentialStorage:(NSURLConnection *)connection
{
    return YES;
}
- (void)connection:(NSURLConnection *)connection
didReceiveAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge
{
    [self.client URLProtocol:self
didReceiveAuthenticationChallenge:challenge];
}
- (void)connection:(NSURLConnection *)connection
didCancelAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge 
{
    [self.client URLProtocol:self
didCancelAuthenticationChallenge:challenge];
}
- (void)connection:(NSURLConnection *)connection
didReceiveResponse:(NSURLResponse *)response
{
    [self.client URLProtocol:self
          didReceiveResponse:response
          cacheStoragePolicy:[[self request] cachePolicy]];
}
- (void)connection:(NSURLConnection *)connection
    didReceiveData:(NSData *)data
{
    [self.client URLProtocol:self
                 didLoadData:data];
}
- (NSCachedURLResponse *)connection:(NSURLConnection *)connection
                  willCacheResponse:(NSCachedURLResponse *)cachedResponse
{
    return cachedResponse;
}
- (void)connectionDidFinishLoading:(NSURLConnection *)connection
{
    [self.client URLProtocolDidFinishLoading:self];
}
```
在每个delgate的实现中我都刨去了工程中的特定实现(流量统计)，只保留了需要实现的最小Protocol集合。

### NSURLProtocol那些坑
从上面的介绍来看，NSURLProtocol还是比较简单，但是实际使用的过程中却容易掉进各种坑，一方面是文档不够详尽，另一方面也是对于苹果这套URL Loading Sytem并不熟悉，不能将整个调用过程有机地统一。

1) 坑1：企图在`canonicalRequestForRequest:`进行request的自定义操作，导致各种递归调用导致连接超时。这个API的表述其实很暧昧:
> It is up to each concrete protocol implementation to define what “canonical” means. A protocol should guarantee that the same input request always yields the same canonical form.

所谓的canonical form到底是什么呢？而围观了包括`NSEtcHosts`和`RNCachingURLProtocol`在内的实现，它们都是直接返回当前request。在这个方法内进行request的修改非常容易导致递归调用(即使通过`setProperty:forKey:inRequest:`对请求打了标记)

2) 坑2：没有实现足够的回调方法导致各种奇葩问题。如`connection:willSendRequest:redirectResponse: `内如果没有通过`[self client]`回传消息，那么需要重定向的网页就会出现问题:host不对或者造成跨域调用导致资源无法加载。


### 同步AFNetworking请求
虽然Mattt各种鄙视同步做网络请求，但是我们不可否认某些场景下使用同步调用会带来不少便利。一种比较简单的实现是使用信号量做同步：
```
@implementation AFHTTPRequestOperation (YX)
- (void)yxStartSynchronous
{
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    [self setCompletionBlockWithSuccess:^(AFHTTPRequestOperation *operation, id responseObject) {
        dispatch_semaphore_signal(semaphore);
    } failure:^(AFHTTPRequestOperation *operation, NSError *error) {
        dispatch_semaphore_signal(semaphore);
    }];
    [self start];
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
}
@end
```

但是这样带来的问题是在UI线程调用同步请求就会导致线程堵死崩溃(好吧，就不应该允许UI线程上这么做)。一种改进的方法是使用NSRunLoop
即：
```
while (_shouldBlock)
{
    [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode
                             beforeDate:[NSDate distantFuture]];
}
```

但是这种写法是有大坑的：如果当前`NSRunLoop`并没有任何NSTimer或`Input Source`，`runMode:beforeDate:`方法将立刻返回NO，于是造成死循环，占用大量CPU，进而导致`NSURLConnection`请求超时。 

规避的方法是往RunLoop中添加NSTimer或者空NSPort使得NSRunLoop挂起而不占用CPU。(ASIHttpRequest就是在当前RunLoop中添加了0.25秒触发一次的刷新Timer)

> If no input sources or timers are attached to the run loop, this method exits immediately and returns NO; otherwise, it returns after either the first input source is processed or limitDate is reached. Manually removing all known input sources and timers from the run loop does not guarantee that the run loop will exit immediately. OS X may install and remove additional input sources as needed to process requests targeted at the receiver’s thread. Those sources could therefore prevent the run loop from exiting.


