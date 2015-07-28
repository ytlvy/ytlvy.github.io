title: Chun notes
categories: IOS
tags:
  - Swift
date: 2015-07-28 21:42:26
author:
description:
photos:
---
[转自](http://blog.callmewhy.com/2015/05/25/note-about-chun/)

## Chun 阅读笔记 - 如何做一个图片缓存库

[Chun](https://github.com/yechunjun/Chun) 是 [叶纯俊](http://chun.tips/) 在 Github 上开源的一个图片缓存库，基于 Swift 编写。学习 Swift 有一段时间了，记录一些阅读源码的一些收获。


### 代码组织
Swift 中通过 `extension` 组织代码会让整个类更加清晰可读，尤其是对于 `UITableViewDataSource` 和 `UITableViewDelegate` 这种情况。在 Chun 这个项目中的 Demo 文件就是这样的：
```
class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        ...
    }
}

extension ViewController: UITableViewDataSource, UITableViewDelegate {
    func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        ...
    }
    func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
        ...
    }
    func tableView(tableView: UITableView, heightForRowAtIndexPath indexPath: NSIndexPath) -> CGFloat {
        ...
    }
}
```

<!-- more -->

在 `viewDidLoad` 中，为了避免初始化代码过长导致难以阅读，可以通过内嵌函数将代码分段：

```
override func viewDidLoad() {
    super.viewDidLoad()

    func loadTableView() {
        ...
    }

    loadTableView()
}
```

### 添加属性
在给 `UIImageView` 加载图片的时候，我们最好可以在对象中存储它所要加载的 URL ，可以通过 `AssociatedObject` 来实现。在 Swift 中，可以用一个私有计算量来封装一下：

```
private var imageURLForChun: NSURL? {
    get {
        return objc_getAssociatedObject(self, &key) as? NSURL
    }
    set (url) {
        objc_setAssociatedObject(self, &key, url, UInt(OBJC_ASSOCIATION_RETAIN_NONATOMIC))
    }
}
```

这样在调用的时候就和真实属性没什么区别了：
```
if let imageURL = self.imageURLForChun {
    ...
}
```

### weak 和 unowned

在避免循环强引用的时候，如果某些时候引用没有值，那就用 `weak` ，如果引用总是有值，则用 `unowned` 。

在 Chun 这个项目中，获取图片之后的回调里用的是 `weak` ，因为有可能图片加载完了但是 `UIImageView` 已经销毁了：
```
Chun.sharedInstance.fetchImageWithURL(url, complete: { [weak self](result: Result) -> Void in
    ...
})
```

然后在查询本地缓存的时候，用的是 `unowned` ，因为这里的 `self` 是单例，永远不会销毁：
```
cache.diskImageExistsWithKey(key, completion: { [unowned self](exist: Bool, diskURL: NSURL?) -> Void in
    ...
})
```

### 枚举的正确打开方式

使用枚举来表示返回结果是个不错的方案，在面向轨道编程 - Swift 中的异常处理中有过详细的探讨。在 Chun 中是这样使用的：
```
public enum Result {
    case Success(image: UIImage, fetchedImageURL: NSURL)
    case Error(error: NSError)
}
```

加载图片完成之后的回调则是这样：
```
public func fetchImageWithURL(url: NSURL, complete: (Result) -> Void) {
    let key = cacheKeyForRemoteURL(url)
    if let image = cache.imageForMemeoryCacheWithKey(key) {
        let result = Result.Success(image: image, fetchedImageURL: url)
        complete(result)
    } else {
        ...
    }
}
```


### 图片渲染
直接从网上下载获取到的图片并不能直接使用，先解码成位图然后再渲染可以减少开销：
```
func decodedImageWithImage(image: UIImage) -> UIImage {
    if image.images != nil {
        return image
    }
    let imageRef = image.CGImage
    let imageSize: CGSize = CGSizeMake(CGFloat(CGImageGetWidth(imageRef)), CGFloat(CGImageGetHeight(imageRef)))
    let imageRect = CGRectMake(0, 0, imageSize.width, imageSize.height)
    let colorSpace = CGColorSpaceCreateDeviceRGB()

    let originalBitmapInfo = CGImageGetBitmapInfo(imageRef)
    let alphaInfo = CGImageGetAlphaInfo(imageRef)

    var bitmapInfo = originalBitmapInfo
    switch (alphaInfo) {
    case .None:
        bitmapInfo &= ~CGBitmapInfo.AlphaInfoMask
        bitmapInfo |= CGBitmapInfo(CGImageAlphaInfo.NoneSkipFirst.rawValue)
    case .PremultipliedFirst, .PremultipliedLast, .NoneSkipFirst, .NoneSkipLast:
        break
    case .Only, .Last, .First:
        return image
    }

    if let context = CGBitmapContextCreate(nil, CGImageGetWidth(imageRef), CGImageGetHeight(imageRef), CGImageGetBitsPerComponent(imageRef), 0 , colorSpace, bitmapInfo) {
        CGContextDrawImage(context, imageRect, imageRef)
        let decompressedImageRef = CGBitmapContextCreateImage(context)
        if let decompressedImage = UIImage(CGImage: decompressedImageRef, scale: image.scale, orientation: image.imageOrientation) {
            return decompressedImage
        } else {
            return image
        }
    } else {
        return image
    }
}
```


### 从 NSData判断图片类型
在判断图片格式的时候，通过不同格式的第一个字节进行判断，在 `contentTypeForImageData(data: NSData) -> String?` 方法里实现了获取 NSData 类型的方法：

```
func contentTypeForImageData(data: NSData) -> String? {
    var value : Int16 = 0
    if data.length >= sizeof(Int16) {
        data.getBytes(&value, length:1)
        switch (value) {
        case 0xff:
            return "image/jpeg"
        case 0x89:
            return "image/png"
        case 0x47:
            return "image/gif"
        case 0x49:
            return "image/tiff"
        case 0x4D:
            return "image/tiff"
        case 0x52:
            if (data.length < 12) {
                return nil
            }
            if let temp = NSString(data: data.subdataWithRange(NSMakeRange(0, 12)), encoding: NSASCIIStringEncoding) {
                if (temp.hasPrefix("RIFF") && temp.hasSuffix("WEBP")) {
                    return "image/webp"
                }
            }
            return nil
        default:
            return nil
        }
    }
    else {
        return nil
    }
} 
```

判断的依据是不同图片格式的前几个字节都是特殊且唯一的，具体在 [File magic numbers](http://www.astro.keele.ac.uk/oldusers/rno/Computing/File_magic.html) 里有个比较完整的表，可以对照看下。比如 jpeg 的前四个字节都是 ff d8 ff e0 。

### Fetcher 的玩儿法
在获取图片的时候都是通过 `Fetcher` 获取，根据任务不同，区分是从服务器下载还是从本地加载。

首先是 `ImageFetcher` 这个大基类，封装了一些基本的属性和方法：

```
lass ImageFetcher {

    typealias CompeltionClosure = (FetcherResult) -> Void

    let imageURL: NSURL

    init(imageURL: NSURL) {
        self.imageURL = imageURL
    }

    deinit {
        self.completion = nil
    }

    var cancelled = false

    var completion: CompeltionClosure?

    static func fetchImage(url: NSURL, completion: CompeltionClosure?) -> ImageFetcher {
        var fetcher: ImageFetcher
        if url.fileURL {
            fetcher = DiskImageFetcher(imageURL: url)
        } else {
            fetcher = RemoteImageFetcher(imageURL: url)
        }
        fetcher.completion = completion
        fetcher.startFetch()
        return fetcher
    }

    func cancelFetch() {
        self.cancelled = true
    }

    func startFetch() {
        fatalError("Subclass need to override this method called: \"startFetch\" ")
    }

    final func failedWithError(error: NSError) {
    }

    final func succeedWithData(imageData: NSData) {
    }
}
```

在 `fetchImage` 这个方法里，通过 `url.fileURL` 判断是网络请求还是本地请求，然后初始化不同的 `fetcher` 。然后对于一定需要子类实现的方法，用 `fatalError` 报错提醒；对于一定不能让子类重写的方法，用 `final` 保护起来。比如请求成功之后的回调方法 `succeedWithData(imageData: NSData) ：`

```
final func succeedWithData(imageData: NSData) {

    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), { [weak self]() -> Void in
        if let strongSelf = self {
            var finalImage: UIImage!

            if let image = imageWithData(imageData) {
                finalImage = scaledImage(image)
                finalImage = decodedImageWithImage(finalImage)
                dispatch_main_async_safe {
                    if !strongSelf.cancelled {
                        if let completionClosure = strongSelf.completion {
                            let result = FetcherResult.Success(image: finalImage, imageData: imageData)
                            completionClosure(result)
                        }
                    }
                }
            } else {
                let error = NSError(domain: CHUN_ERROR_DOMAIN, code: 404, userInfo: [NSLocalizedDescriptionKey: "create Image with data failed"])
                strongSelf.failedWithError(error)
            }
        }
    })
}
```

不管是从本地加载还是从远程获取的，最终的返回结果都是 `NSData` ，所以在这里统一处理。然后对于取消了的事件，其实并没有取消下载任务，而是在下载成功之后通过 `strongSelf.cancelled` 判断是不是要调用加载成功的回调方法。

然后再分别看下本地加载和网络获取的部分。本地加载相对而言简单一些，通过 `NSData(contentsOfURL: self.imageURL)`就可以加载图片了。然后对于网络请求则使用了 `NSURLSession` 来实现。 对 `NSURLSession` 不熟悉的同学可以阅读《从 NSURLConnection 到 NSURLSession》了解一下。

网络请求成功之后做了如下操作：

- 检查 `self` 是否还活着
- 检查当前任务是否被取消了
- 检查回调的 `error` 是否不为空
- 获取 `response` 并查看状态码是否为 `200`

在一切正常的前提下，还进行了如下操作：
```
let expected = response.expectedContentLength
var validateLengthOfData: Bool {
    if expected > -1 {
        if Int64(data!.length) >= expected {
            return true
        } else {
            return false
        }
    }
    return true
}

if validateLengthOfData {
    strongSelf.succeedWithData(data!)
    return
} else {

    let error = NSError(domain: CHUN_ERROR_DOMAIN, code: response.statusCode, userInfo: [NSLocalizedDescriptionKey: "Received bytes are not fit with expected"])
    strongSelf.failedWithError(error)
    return
}
```

主要是检查实际获取到的数据大小是否等于应有大小，通过 `validateLengthOfData` 这个计算量标记是否校验通过。

### 缓存
图片的缓存都是通过 `ImageCache` 这个类进行统一处理。初始化的时候新建了 `ioQueue` 这个用来专门进行 `IO` 操作的队列，然后用 `NSCache` 在内存中缓存图片。对于 `NSCache` 在 `NSHipster` 上有些吐槽，但这并没有太大影响，基本可以满足日常开发的需要。

### 系统事件的处理
在收到 `UIApplicationDidEnterBackgroundNotification` 的通知的时候，做了 `backgroundCleanDisk` 的处理：

```
private func backgroundCleanDisk() {

    let application = UIApplication.sharedApplication()
    var backgroundTask: UIBackgroundTaskIdentifier!
    backgroundTask = application.beginBackgroundTaskWithExpirationHandler {
        application.endBackgroundTask(backgroundTask)
        backgroundTask = UIBackgroundTaskInvalid
    }

    self.cleanDisk {
        application.endBackgroundTask(backgroundTask)
        backgroundTask = UIBackgroundTaskInvalid
    }
}
```

通过 `beginBackgroundTaskWithExpirationHandler` 在退到后台之后清空了本地的过期文件。

### 过期文件

判断过期文件的关键在于这个方法：
```
let expirationDate = NSDate(timeIntervalSinceNow: ImageCache.defaultCacheMaxAge)
let modificationDate = resourceValues[NSURLContentModificationDateKey] as! NSDate

if modificationDate.laterDate(expirationDate).isEqualToDate(expirationDate) {
    ...
}
```

通过遍历检查所有的过期文件，存到 cacheFiles 数组中，然后统一删除。

## 小结

通过 Chun 这个项目学习了如何实现一个简单的图片缓存库，包括图片加载和本地缓存两个核心功能。然后通过 public class 把一些公用接口封装并暴露出去。也看到了很多 Swift 中的小技巧，总之就是， Excited 嗯！




