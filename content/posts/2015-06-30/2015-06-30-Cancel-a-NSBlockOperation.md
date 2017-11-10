---
author: null
categories:
- IOS
date: '2015-06-30'
description: null
photos: null
tags:
- IOS
- NSBlockOperation
title: Cancel a NSBlockOperation
---

### make NSBlockOperation cancelable

```
NSBlockOperation *operation = [[NSBlockOperation alloc] init];
__weak NSBlockOperation *weakOperation = operation;
[operation addExecutionBlock:^{
   while( ! [weakOperation isCancelled]){
      //do something...
   }
}];
```