title: iOS Application Framework Think - Network
categories: IOS
tags:
  - Framework
date: 2015-08-05 21:46:31
author:
description:
photos:
---
[reference](http://blog.cnbluebox.com/blog/2015/05/07/architecture-ios-1/)

## IOS应用架构思考一（网络层）

最近看到Casa Taloyum同学的关于IOS架构的文章，分享的概念和观点很值得一看，于是不禁心痒，也做些分享吧，我会从实际设计过程中需要思考的问题的角度着手来讲述，毕竟无论什么样的架构，什么样的设计都是要解决这些问题的。
今天就先讲讲网络层的需要思考的问题吧。

### requestOperation的设计

我们都知道在客户端发送请求是需要成本的，那么设计异步的请求就是首要的问题。我们知道Cocoa提供了非常丰富和易于使用的异步api, 有`NSOperationQueue`, `dispatch queue`, `NSThread`等。那么如何选择呢，答案毫无疑问的必须是`NSOperation`。 不信你去看ASIHTTPRequest和AFNetworking. 好像这个理由不够充分是吧，而且很多人就是使用了这些框架，而不清楚它们到底为何优秀，那我就列举下一些它们的优点吧

首先举个反例吧：曾经有人问过我“为什么要用AFNetworking和ASI这样的框架呢，我直接dispatch到后台用类似dataWithURL:拉数据这样不是很简单？”。 那么这样的使用案例有2个很致命的缺陷：

1. 无法cancel这个请求
2. 占用了完整的一个线程

<!-- more -->

#### 可以cancel
请求可以cancel的需求的重要性不用说了吧，别说你没用过。使用NSOperation可以让你方便的设计cancel一个请求的方法。

#### 线程
说道线程可能很多人对发送请求的时候线程有个很大的误解： 每个请求占用一个线程。其实不是的，无论同时并发多少个请求，AFNetworking和ASI都是只有一个线程在等待的。不信请看AFNetworking的实现：

```
+ (void)networkRequestThreadEntryPoint:(id __unused)object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];

        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}

+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });

    return _networkRequestThread;
}

- (void)start {
    [self.lock lock];
    if ([self isReady]) {
        self.state = AFOperationExecutingState;

        [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    }
    [self.lock unlock];
}
```

从上面的代码中我们可以看到，`AFNetworking`在等待请求时其实只有个一个Thread, 然后在这个Thread上启动一个runLoop监听 `NSURLConnection` 的 `NSMachPort` 类型源。 `start`里面直接就跳到这个线程去执行了， 在加入`NSOperationQueue`时，顶多`start`方法执行的时候占用一个线程，然后真正的发送请求和等待都是在这个`networkRequestThread`里面进行的。

> 补充一下，上面的描述可能引起误会，已经有人问我这个疑问了。注意， 上面讲的networkRequestThread 并不是最终访问网络并拉取数据的线程。真正的下载数据是NSURLConnection统一调度的，networkRequestThread 只是启用了一个runLoop来监听NSURLConnection的回调事件。

了解上面两点后我们再看前面的例子，就会发现问题有多致命了。

> 如果同时并发了很多个请求，那就是实实在在的占用了n个线程了，而且请求不能cancel，对于这些被占用掉得大量资源就束手无策了。

#### 并发数量限制

如果用`NSOperationQueue`来说明，就是`maxConcurrentOperationCount`, 最大并发数量。
可能有人指出了, 刚才不是讲请求都是在一个线程等待的吗，那并发数量岂不是没有太大意义？ NO, 并发数量还是有重要的意义的，它的主要意义就是***控制连接数***。> 这里多谢 CasaTaloyum 同学的勘误，我原本不知道2G, 3G, 等网络协议的限制。

2G网络下一次只能维持一个链接，3G是2个，4G和wifi是不限。这个是对应协议的限制。如果超过这个限制发出的请求，就会报超时。
除掉这个连接数的影响外，还有次要的影响就是带宽，下载数据是要流量的，如果同时并发了太多请求，每个连接都占用带宽，可能导致每个请求的时间均会延长，就好像你在迅雷同时下载10个文件，那么下载速度都慢了。不过考虑到现在的网速，这个场景一般都比较极限，在做一些特殊场景优化的时候或许可以考虑到。

当然`maxConcurrentOperationCount`也不能设置太小，太小了的话，如果个别的请求太慢，导致后面的任务就启动不起来了。
并发数量的考虑应该从上面两个点出发，因此如果有人再扯到线程上去，只能说走远了。

#### pause & dependency
可暂停的，可添加依赖的，这些可能不常会用到，但是如果要设计网络层框架还是要考虑的，这也是要用`NSOperation`的原因之一。

> 本节内容说出了开源框架在operation的设计上的一些优点，因此也推荐这部分直接使用AFNetworking之类的框架，因为优秀的开源组件总有其优点，他们考虑了很多你甚至都没有意识到的问题，以前有是遇到过一些同学质疑AFNetworking,质疑arc, 质疑SDWebImage, 质疑Masonry等，当然不是说没有缺点，但是这些东西当你真正深入了解学习后我觉得才能做出正确的选择。

### 安全性

很多IOS客户端开发者不注重安全性，当然也因为IOS系统已经做得很不错的原因，还有安全的主要工作是在后端，比如使用https啊，比如加签啊，那么前段的网络框架设计要注意哪些安全性的问题呢。

#### 数据加密和防篡改

如无特殊情况，数据加密就推荐用https就够了，好处就不一一列举了，现在各大互联网公司（如BAT）都在做“全站https”. 而且现在免费的SSL证书也非常好申请，几乎不要什么成本了。

要保证接口参数不被篡改，不会被第三方恶意攻击，就要对参数进行签名。记得以前在某国内知名电商，由于接口没有类似防护，一两个小时被人利用移动端的注册接口注册了80w+账号，惨不忍睹。 而且签名密钥（其实不是密钥，就是个干扰串）不要写死，要动态下发。

#### https中间人

https是非常安全的协议，可以参考我以前的博客RSA加密数字证书里面。但是可能还是会存在中间人攻击的问题，那么就需要打ssl钢钉，AFNetworking中的`AFURLConnectionOperationSSLPinningMode`就是可以设置ssl钢钉的，原理就是把证书或者公钥 打包到bundle中，发送请求的时候会与请求过来的证书比较，因此避免中间人发放的伪造证书可能。

#### DNS防劫持
进一步的安全策略就可以考虑到DNS防劫持，原理比较简单，可以本地维护一个路由表，然后本地实现 NSURLProtocol 对host进行ip映射。如果你还不太了解NSURLProtocol，可以参考[NSURLProtocol](http://nshipster.cn/nsurlprotocol/)进行学习

### 组件设计
无论你是用AFNetworking之类的开源组件还是自己封装的方式，一般我们都会封装下中间组件，一来可以降低业务代码和底层组件的耦合，二来方便拓展。那么设计这些组件的时候需要注意哪些呢？

#### 回调方式
这部分选择不少，主要是能统一和方便，可能会考虑下面2个问题 1、是选择block回调方式，还是delegate, 还是target-action, 可以根据自己的喜好来设置，也可以兼容多种。 2、success和fail分开回调还是同一个方法回调。

#### 拦截器

request的部分可能会有些场景需要请求前和请求后的拦截器的设计， 即AOP入口，在这样的拦截器接口中，我们就可以做一些事情如：

1. session过期后自动登录 
2. 需要登录的接口进入登录

#### 统一cancel还是单独cancel
在一个页面中发送请求的时候，无疑会碰到一个问题，那就是“页面退出时请求要取消”。
我们最开始是如何做的呢，就像内存管理，谁启动，谁cancel, 即是单独cancel. 使用方式类似下面：
```
@interface AViewController: UIViewController {
  HTTPRequest *_request;
}
@end
@implementation AViewController
- (void)dealloc {
  [_request cancel];
}
- (void)sendRequest {
  [_request cancel];
  _request = [[HTTPRequest alloc] init];
  [HttpClient sendRequest:_request];
}
@end
```

然后就觉得每个地方都要在dealloc里面写cancel，很麻烦，于是可能想要做个统一的cancel，类似封装个方法在父类里面调用：
```
@implementation BaseViewController
- (void)dealloc {
  [HttpClient cancelRequestForDelegate:self];
}
@end
```
这样的话好像就更方便了，代码量也少了一些，调用的类也不用去管cancel了， 但是这样真的好么，虽然可能让你暂时方便了，但是这样的设计还是不紧凑，其实这个请求是否应该被取消应该是调用者的逻辑，这样就相当于一部分的逻辑放在父类里面，是有点坑的，第一种写法虽然可能麻烦写，但是就像内存管理，谁创建，谁负责销毁，这样的代码可读性和维护性更高些。不然如果遇到特殊的逻辑，可能要去动父类或底层。

> 说到这里就是建议上面代码中的request对象的封装就是Operation本身，这样其实就不需要再在viewController里面写很多isLoading, isLoaded的状态位了啊，NSOperation自带的。

### 缓存

说道网络请求，就不能不提网络缓存，数据缓存是提升用户体验不可缺少的重要一环。而对于缓存可以讲的太多了，比如图片缓存可以单独开篇文章来讲了，这里就讲一下关于普通接口数据的缓存。一般我们对于缓存的方式有两种做法，一种是客户端写缓存实现和缓存逻辑，一种是遵循HTTP协议的缓存。

#### 自己实现缓存逻辑
一般需要这种缓存的逻辑无非是考虑下面的需求：

> 1、接口返回的数据很少变动，不希望做重复请求 2、网络慢或者服务器等异常状况容灾。

自己实现也比较灵活，你可以写个数据库来缓存、也可以直接序列化存文件。但是缺点也比较明显，那就是缓存时间不好定义。

既然做了缓存，那么就会遇到这个问题，就是数据刷新了之后，缓存不更新怎么办。那么其实在你选择这种缓存实现之前就需要做权衡，是数据实时性重要，还是数据加载速度重要。想好这个问题才能确定是否要这样的缓存。

#### HTTP缓存
这种缓存方案是比较推荐的方案，HTTP协议已经定义了好了一套缓存方案，为什么不用呢。 而且苹果已经帮我们实现好了缓存的代码，NSURLCache，数据缓存在本地sqlite里。不过就是要和服务端同学一块推进。

可以缓存的接口，可以在返回的 responseHeaders 中添加 cache-control 和 Expires 来告诉前端是否可以缓存和缓存时间等。

而客户端可以通过设置 cachePolicy 来决定是否使用缓存
```
typedef NS_ENUM(NSUInteger, NSURLRequestCachePolicy)
{
    NSURLRequestUseProtocolCachePolicy = 0,

    NSURLRequestReloadIgnoringLocalCacheData = 1,
    NSURLRequestReloadIgnoringLocalAndRemoteCacheData = 4, // Unimplemented
    NSURLRequestReloadIgnoringCacheData = NSURLRequestReloadIgnoringLocalCacheData,

    NSURLRequestReturnCacheDataElseLoad = 2,
    NSURLRequestReturnCacheDataDontLoad = 3,

    NSURLRequestReloadRevalidatingCacheData = 5, // Unimplemented
};
```

> 考虑一种场景，你不能确定数据什么时候更新，可能随时刷新，也可能几天才更新，那么我们如何使用缓存呢。
可以使用 `ETag/If-None-Match` 或者 `Last-Modified/If-Modified-Since`。

两种方式原理差不多，就是在发送请求的 requestHeaders 中加入 `If-None-Match` 或者 `If-Modified-Since` 字段，和服务端数据进行比较是否有更新，有更新则返回body，没有的话就在responseHeaders中告知使用缓存。 两者中， ETag是比较hash， Last-Modified是比较最后更改时间。

ASIHTTPRequest实现了这种模式，如下
```
if ([self cachePolicy] & (ASIAskServerIfModifiedWhenStaleCachePolicy|ASIAskServerIfModifiedCachePolicy)) {

  NSDictionary *cachedHeaders = [[self downloadCache] cachedResponseHeadersForURL:[self url]];
  if (cachedHeaders) {
      NSString *etag = [cachedHeaders objectForKey:@"Etag"];
      if (etag) {
          [[self requestHeaders] setObject:etag forKey:@"If-None-Match"];
      }
      NSString *lastModified = [cachedHeaders objectForKey:@"Last-Modified"];
      if (lastModified) {
          [[self requestHeaders] setObject:lastModified forKey:@"If-Modified-Since"];
      }
  }
}
```

### 服务器推

对于很多需要精细化体验的app来说，HTTP协议的模式已经不能够满足需求了， 比如支付宝可以随时随地的推活动。那么就需要服务器推的技术了。

#### SPDY 或 HTTP/2

spdy 和 HTTP2 协议都定义了关于服务端推送的协议，还有一些其他的相对于http的优良特性. 一些大公司应该都做了对于spdy的支持，ios中可以使用twitter开源的CocoaSPDY。 不过Google刚刚宣布了不再支持SPDY，以后都走HTTP/2了。

关于这些我还没有详细的学习，这里就不多讲了，以后多多学习。

#### 长连接
一种比较灵活和强大的方法服务器推送的方案就是TCP长连接了，长连接可以让你做很多事情，唯一的难题就是要处理好心跳包。

### 其他

####  国际化
在做国际化的时候，需要向服务器提供用户的语言偏好，也就是需要设置 Accept-Language 字段，如下：
```
[request setValue:[NSString stringWithFormat:@"%@", [[NSLocale preferredLanguages] componentsJoinedByString:@", "]], forHTTPHeaderField:@"Accept-Language"];
```

即使你的服务器还不支持国际化，把这个属性放进去会让你在需要的时候用到，而不必更新客户端。`AFNetworking`已经实现了改逻辑。




