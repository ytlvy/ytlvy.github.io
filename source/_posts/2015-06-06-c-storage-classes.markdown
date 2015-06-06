---
layout: post
title: "C Storage Classes"
date: 2015-06-06 20:49:35 +0800
comments: true
categories: C
---
## C Storage Classes
There are 4 storage classes in C: auto, register, static & extern.

### static
When it comes to storage classes, static means one of two things.
1. A static variable inside a method or function retains its value between invocations.
2. A static variable declared globally can called by any function or method, so long as those functions appear in the same file as the static variable. The same goes for static functions.

### Static Singletons
```
+ (instancetype)sharedInstance{
    static id _sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispath_once(&onceToken, ^{
        _sharedInstance = [[self alloc] init];
        //any config
        })
    return _sharedInstance;
}
```

<!--more-->

### extern
Whereas static makes functions and variables globally visible within a particular file, extern makes them visible globally to all files.

#### Global String Constants
Any time your application uses a string constant in a public interface, it should be declared as an external string constant. This is especially true of NSNotification names, NSError domains, and keys in userInfo dictionaries.
Declare an extern NSString * const in a public header, and define that NSString * const in the implementation:

AppConstants.h
```
extern NSString * const kAppErrorDomain; 
```

AppConstants.m
```
NSString * const kAppErrorDomain = @"com.example.yourapp.error";
```

#### Public Functions
TransactionStateMachine.h
```
typedef NS_ENUM(NSUInteger, TransactionState) {
    TransactionOpened,
    TransactionPending,
    TransactionClosed,
}!;
extern NSString * NSStringFromTransactionState(TransactionState state); 
```

TransactionStateMachine.m
```
NSString *
  NSStringFromTransactionState(TransactionState state) {
  switch (state) {
    case TransactionOpened: return @"Opened"
    case TransactionPending: return @"Pending";
    case TransactionClosed: return @"Closed";
    default: return nil;
  } 
}
```

