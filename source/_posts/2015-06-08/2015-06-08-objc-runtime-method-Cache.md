title: objc runtime method Cache
categories: IOS
tags:
  - runtime 
  - cache
description:
photos:
date: 2015-06-08 21:01:55
author: GYJ
---
### method cache 更新机制
为了方便函数的查找, objc在运行时,引入了method cache机制,来快速定位函数.method cache的大小是动态更新的,在更新时, 将直接生成新容量的缓存, 并将旧的缓存放入到垃圾队列中,等待时机释放. 而时机的选择, 采用检测每个线程的program counter,来查看是否处于objc_msgSend.如果都不在,则释放.
```
BOOL ThreadsInMsgSend(void) {
    for(thread in GetAllThreads()) {
        uintptr_t pc = thread.GetPC();
        if(pc >= objc_msgSend_startAddress && pc <= objc_msgSend_endAddress) {
            return YES;
        }
    }
    return NO;
}

bucket_t *oldCache = class->cache;
class->cache = malloc(newSize);

append(gOldCachesList, oldCache);
if(!ThreadsInMsgSend()) {
    for(cache in gOldCachesList) {
        free(cache);
    }
    gOldCachesList.clear();
}
```


