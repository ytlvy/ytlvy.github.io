---
layout: post
title: "ios nil  Nil NULL NSNull"
date: 2015-06-05 10:23:26 +0800
comments: true
categories: IOS
---

## nil Nil NULL NSNull

> C represents nothing as 0 for primitive values, and NULL for pointers (which is equivalent to 0 in a pointer context)
Objective-C builds on C's representation of nothing by adding nil. nil is an object pointer to nothing. Although semantically distinct from NULL, they are equivalent to one another.
On the framework level, Foundation defines the NSNull class, which defines a single class method, +null, which returns a singleton NSNull instance. NSNull is different from nil or NULL, in that it is an actual object, rather than a zero value.
Additionally, in Foundation/NSObjCRuntime.h, Nil is defined as a class pointer to nothing. This lesser-known title- case cousin of nil doesn't show up much very often, but it's at least worth noting.

<!--more-->

### NSNull Something for Nothing
NSNull is used throughout Foundation and other system frameworks to skirt around the limitations of collections like NSArray and NSDictionary, which cannot contain nil values. NSNull effectively boxes NULL or nil values, so that they can be stored in collections:

```
NSMutableDictionary *mutableDictionary = [NSMutableDictionary dictionary];
mutableDictionary[@"someKey"] = [NSNull null];
NSLog(@"Keys: %@", [mutableDictionary allKeys]); 
```

### representing nothing
Symbol    | Value            | Meaning
:---------|:---------------- |--------
NULL      |(void *)0         | literal null value for C pointers
nil       |(id)0             | literal null value for Objective-C objects
Nil       |(Class)0          | literal null value for Objective-C classes
NSNull    |[NSNull null]     | singleton object used to represent null
