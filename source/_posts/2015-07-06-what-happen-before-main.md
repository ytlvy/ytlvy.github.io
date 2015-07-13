title: what happen before main
categories: IOS
tags:
  - IOS
  - dyld

description:
date: 2015-07-06 21:20:25
author:
photos:
---

[转自](http://blog.sunnyxx.com/2014/08/30/objc-pre-main/)

## iOS程序main函数之前发生了什么
### 前言
一个iOS app的`main()`函数位于`main.m`中，这是我们熟知的程序入口。但对objc了解更多之后发现，程序在进入我们的main函数前已经执行了很多代码，比如熟知的`+ load`方法等。本文将跟随程序执行顺序，刨根问底，从`dyld`到`runtime`，看看main函数之前都发生了什么。


### 从dyld开始
#### 动态链接库
iOS中用到的所有系统`framework`都是动态链接的，类比成插头和插排，静态链接的代码在编译后的静态链接过程就将插头和插排一个个插好，运行时直接执行二进制文件；而动态链接需要在程序启动时去完成“插插销”的过程，所以在我们写的代码执行前，动态连接器需要完成准备工作。

这个是在xcode中看到的Link列表：
![](http://ww4.sinaimg.cn/mw600/51530583jw1ejx4ul5susj20wg0803zh.jpg)
<!-- more -->
这些`framework`将会在动态链接过程中被加载，另外还有隐含`link`的`framework`，可以测试出来：先找到可执行文件，我这里叫TestMain的工程，模拟器路径下找到`TestMain.app`，可执行文件默认同名，再通过`otool`命令：
```
$ otool -L TestMain
```
-L参数打印出所有link的framework
```
TestMain:
    /System/Library/Frameworks/CoreGraphics.framework/CoreGraphics 
    /System/Library/Frameworks/UIKit.framework/UIKit
    /System/Library/Frameworks/Foundation.framework/Foundation
    /System/Library/Frameworks/CoreFoundation.framework/CoreFoundation 
    /usr/lib/libobjc.A.dylib 
    /usr/lib/libSystem.dylib
```

除了多了的`CoreGraphics`（被`UIKit`依赖）外，有两个默认添加的`lib`。`libobjc`即`objc`和`runtime`，`libSystem`中包含了很多系统级别`lib`，列几个熟知的：`libdispatch(GCD)`，`libsystem_c(C语言库)`，`libsystem_blocks(Block)`，`libcommonCrypto`(常用的md5函数)等等。这些lib都是dylib格式（如windows中的dll），系统使用动态链接有几点好处：

- 代码共用：很多程序都动态链接了这些lib，但它们在内存和磁盘中中只有一份
- 易于维护：由于被依赖的lib是程序执行时才link的，所以这些lib很容易做更新，比如`libSystem.dylib`是`libSystem.B.dylib`的替身，哪天想升级直接换成`libSystem.C.dylib`然后再替换替身就行了
- 减少可执行文件体积：相比静态链接，可执行文件的体积要小很多

### dyld
`dyld` - `the dynamic link editor` apple的动态链接器，系统`kernel`做好启动程序的初始准备后，交给dyld负责，援引并翻译[《mikeask这篇blog》](https://www.mikeash.com/pyblog/friday-qa-2012-11-09-dyld-dynamic-linking-on-os-x.html)对dyld作用顺序的概括


1. 从kernel留下的原始调用栈引导和启动自己
2. 将程序依赖的动态链接库递归加载进内存，当然这里有`缓存机制`
3. non-lazy符号立即link到可执行文件，lazy的存表里
4. Runs static initializers for the executable
5. 找到可执行文件的main函数，准备参数并调用
6. 程序执行中负责绑定lazy符号、提供runtime dynamic loading services、提供调试器接口
7. 程序main函数return后执行static terminator
8. 某些场景下main函数结束后调libSystem的_exit函数


得益于dyld是开源的，![github地址](https://github.com/opensource-apple/dyld)，我们可以从源码一探究竟。

一切源于`dyldStartup.s`这个文件，其中用汇编实现了名为`__dyld_start`的方法，汇编太生涩，它主要干了两件事：

1. 调用dyldbootstrap::start()方法（省去参数）
2. 上个方法返回了main函数地址，填入参数并调用main函数

这个步骤随手就能验证出来，设置一个符号断点断在_objc_init：
![](http://ww1.sinaimg.cn/mw600/51530583jw1ejxgn8un3cj20oo09675i.jpg)

这个函数是`runtime`的初始化函数，后面会提到。程序运行在很早的时候断住，这时候看调用栈：

![](http://ww3.sinaimg.cn/mw600/51530583jw1ejxgwiptytj20jw0f0q5r.jpg)

看到了栈底的`dyldbootstrap::start()`方法，继而调用了`dyld::_main()`方法，其中完成了刚才说的递归加载动态库过程，由于`libSystem`默认引入，栈中出现了`libSystem_initializer`的初始化方法。

### ImageLoader
当然这个`image`不是图片的意思，它大概表示一个二进制文件（可执行文件或so文件），里面是被编译过的符号、代码等，所以`ImageLoader`作用是将这些文件加载进内存，且每一个文件对应一个I`mageLoader`实例来负责加载。
两步走：

1. 在程序运行时它先将动态链接的image递归加载 （也就是上面测试栈中一串的递归调用的时刻）
2. 再从可执行文件image递归加载所有符号

当然所有这些都发生在我们真正的main函数执行前。

------

### runtime与+load
刚才讲到`libSystem`是若干个系统`lib`的集合，所以它只是一个容器`lib`而已，而且它也是开源的，里面实质上就一个文件，`init.c`，细节不说了，由`libSystem_initializer`逐步调用到了`_objc_init`，这里就是objc和runtime的初始化入口。

除了runtime环境的初始化外，_objc_init中绑定了新image被加载后的callback：
```
dyld_register_image_state_change_handler(dyld_image_state_bound,
                                             1/*batch*/, &map_images);
dyld_register_image_state_change_handler(dyld_image_state_dependents_initialized, 0/*not batch*/, &load_images);

```
可见dyld担当了`runtime`和`ImageLoader`中间的协调者，当新`image`加载进来后交由`runtime`大厨去解析这个二进制文件的符号表和代码。继续上面的断点法，断住神秘的+load函数：
![](http://ww1.sinaimg.cn/mw690/51530583jw1ejyjgvetq1j20jk0bc0uf.jpg)

清楚的看到整个调用栈和顺序：
1. dyld开始将程序二进制文件初始化
2. 交由ImageLoader读取image，其中包含了我们的类、方法等各种符号
3. 由于runtime向dyld绑定了回调，当image加载到内存后，dyld会通知runtime进行处理
4. runtime接手后调用map_images做解析和处理，接下来load_images中调用call_load_methods方法，遍历所有加载进来的Class，按继承层级依次调用Class的load方法和其Category的load方法

至此，可执行文件中和动态库所有的符号（Class，Protocol，Selector，IMP，…）都已经按格式成功加载到内存中，被runtime所管理，再这之后，runtime的那些方法（动态添加Class、方法混合等等才能生效）

#### 关于load方法的几个QA
Q: 重载自己Class的load方法时需不需要调父类？
A: runtime负责按继承顺序递归调用，所以我们不能调super

Q: 在自己Class的load方法时能不能替换系统framework（比如UIKit）中的某个类的方法实现
A: 可以，因为动态链接过程中，所有依赖库的类是先于自己的类加载的

Q: 重载load时需要手动添加@autoreleasepool么？
A: 不需要，在runtime调用load方法前后是加了objc_autoreleasePoolPush()和objc_autoreleasePoolPop()的。

Q: 想让一个类的load方法被调用是否需要在某个地方import这个文件
A: 不需要，只要这个类的符号被编译到最后的可执行文件中，load方法就会被调用（Reveal SDK就是利用这一点，只要引入到工程中就能工作）

-------

## 简单总结
整个事件由dyld主导，完成运行环境的初始化后，配合ImageLoader将二进制文件按格式加载到内存， 动态链接依赖库，并由runtime负责加载成objc定义的结构，所有初始化工作结束后，dyld调用真正的main函数。

值得说明的是，这个过程远比写出来的要复杂，这里只提到了runtime这个分支，还有像GCD、XPC等重头的系统库初始化分支没有提及（当然，有缓存机制在，它们也不会玩命初始化），总结起来就是main函数执行之前，系统做了茫茫多的加载和初始化工作，但都被很好的隐藏了，我们无需关心。

### 孤独的main函数
当这一切都结束时，dyld会清理现场，将调用栈回归，只剩下：
![](http://ww3.sinaimg.cn/mw690/51530583jw1ejykutdlvsj20fc02smx9.jpg)

孤独的main函数，看上去是程序的开始，确是一段精彩的终结


### References

https://www.mikeash.com/pyblog/friday-qa-2012-11-09-dyld-dynamic-linking-on-os-x.html
http://newosxbook.com/articles/DYLD.html
http://docstore.mik.ua/orelly/unix3/mac/ch05_02.htm
https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/dyld.1.html

