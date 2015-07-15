title: Cancel a NSBlockOperation
categories: IOS
tags:
  - IOS
  - NSBlockOperation
description:
date: 2015-06-30 09:08:23
author:
photos:
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
