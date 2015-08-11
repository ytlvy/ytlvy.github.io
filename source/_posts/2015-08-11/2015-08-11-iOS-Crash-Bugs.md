title: iOS Crash Bugs
categories: IOS
tags:
  - Bug
date: 2015-08-11 22:12:12
---

## Crash Bug
1) ios7 下没有 `ContainString` API

2) NSDictionary & NSArray nil insert 

3) 数组越界 NSArray `out of bounds`

4) `substringwithrange` out of bounds

5) `GPUImageView presentBufferForDisplay` -- 内存泄露 或者 app 进入后台

6) `locationOfTouch: inView:` 

```
if (gestureRecognizer)numberOfTouches > 0){

}
```

<!-- more -->

7) "[NSPlaceholderMutableString initWithString:]: nil argument" - NSString的`initWithString:`或者`stringWithString:` 传入了 nil 参数

8) 调用为 nil 的block
```
if (block) {
    block();    //确定不为空之后才放心地调用
}
```

9) 调用了不存在的方法
```
if ([a respondsToSelector:@selector(aaa)]) {
    [a aaa];            //确定有该方法之后才放心地调用
}
```

10) 在cellForRowAtIndexPath中返回了nil

出现这种情况的原因有：

`numberOfRowsInSection`返回的数目不正确，导致行数比`cellForRowAtIndexPath`预期的多，于是`cellForRowAtIndexPath`就不能正确返回超出预期的cell了。
`cellForRowAtIndexPath`中逻辑有误，漏了一些情况，导致有些cell不能正确返回。

11) NSJSONSerialization 识别失败 为 NSNull 