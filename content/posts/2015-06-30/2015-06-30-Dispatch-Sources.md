---
author: null
categories:
- IOS
date: '2015-06-30'
description: null
photos: null
tags:
- IOS
- DISPATCH_SOURCE
title: Dispatch Sources
---

## Dispatch Sources
简单来说，是一种能够监控某种事件的对象。当事件发生的时候，此对象自动唤醒设置好的block，并在指定的queue中运行。


### events Type
- Mach port send right state changes.
- Mach port receive right state changes.
- External process state change.
- File descriptor ready for read.
- File descriptor ready for write.
- Filesystem node event.
- POSIX signal.
- Custom timer.
- Custom event.

### Custom Events
通过`dispatch_source_merge_data`来发送消息，此方法取名`merge`的原因是，在事件回调执行前，GCD会自动合并累计的消息，直到对应的运行Queue有空闲，可以运行回调Block。所以，这是一种提高效率的方式，将多次消息合并成一个。
自定义事件分为：`DISPATCH_SOURCE_TYPE_DATA_ADD` 和 `DISPATCH_SOURCE_TYPE_DATA_OR`, 每个event Source有一个`unsigned long data`属性，用来合并`dispatch_source_merge_data`的参数。`dispatch_source_get_data`可以获取到当前的`data`数据。

<!--more-->

### 举例
异步更新progressBar的状态
```
dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_DATA_ADD, 
                                                0, 
                                                0,
                                                dispatch_get_main_queue());
dispatch_source_set_event_handler(source, ^{
    [progressIndicator incrementBy:dispatch_source_get_data(source)];
    });

dispatch_apply([array count], globalQueue, ^{
    dispatch_source_merge_data(source, 1);
    });
```