---
layout: post
title: "[译]BSS数据段的意义"
date: 2015-05-27 15:54:32 +0800
comments: true
categories: [IOS, Base]
---
摘录：http://stackoverflow.com/questions/9535250/why-is-the-bss-segment-required

## 一个典型的 C 程序内存特征将会有如下几个区域：
> * 文本段（Text segment）
> * 已初始化数据段（Initialized data segment）
> * 未初始化数据段（Uninitialized data segment）
> * 栈（Stack）
> * 堆（Heap）

![](http://d2o58evtke57tz.cloudfront.net/wp-content/uploads/Memory-Layout-300x255.gif)

### 

```
#include <stdio.h>
#include <stdlib.h>


//初始化 存放在`已初始化数据段`也叫data段
int a[10] = { 1, 2, 3, 4, 5, 6, 7, 8, 9};
int b[20]; /* 未初始化 存放在 BSS数据段 */

int main ()
{
   ;
} 
```


### BSS数据段的意义---减少程序大小
在嵌入式系统中，数据和常量存储在真实的ROM（闪存）中。嵌入式系统启动时，首先要设置好所有静态存贮持久对象，然后才会调用main函数。伪代码如下：

```
for(i=0; i<all_explicitly_initialized_objects; i++)
{
  .data[i] = init_value[i];
}

memset(.bss,  0,  all_implicitly_initialized_objects);
```
此时，data段和bss段存储于RAM，而init_value则存贮于ROM。如果只有一个数据段的话，ROM中将存贮好多0，从而增加ROM的大小。

### 效率
由于BSS段的存在，在设置初始化数据时，可以线性操作，从而避免了需要到ROM中取init_value的操作，节省时间。

memset 执行效率高
 