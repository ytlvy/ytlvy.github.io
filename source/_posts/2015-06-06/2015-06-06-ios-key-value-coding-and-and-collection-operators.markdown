---
layout: post
title: "IOS Key-Value Coding &amp;&amp; Collection Operators"
date: 2015-06-06 20:52:04 +0800
comments: true
categories: IOS
---
## Key-Value Coding && Collection Operators
> KVC Collection Operators allows actions to be performed on a collection using key path notation in valueForKeyPath:. @'s in a key path denote an aggregate function whose result can be returned or chained, just like any other key path.

### Simple Collection Operators
1. @count: Returns the number of objects in the collection.
2. @sum: Converts each object in the collection to a double, computes the sum, and returns the sum.
3. @avg: Takes the double value of each object in the collection, and returns the average value.
4. @max: Determines the maximum value using compare:. Objects must support mutual comparison for this to work.
5. @min: Same as @max, but returns the minimum value.

<!--more-->

```
[products valueForKeyPath:@"@count"]; // 4
[products valueForKeyPath:@"@sum.price"]; // 3526.00
[products valueForKeyPath:@"@avg.price"]; // 881.50
[products valueForKeyPath:@"@max.price"]; // 1699.00
[products valueForKeyPath:@"@min.launchedOn"];
```

```
To get the aggregate value of an array or set of NSNumbers, simply pass self as the key path after the operator,
e.g. [@[@(1), @(2), @(3)] valueForKeyPath:@"@max.self "]
```

### Object Operators
@unionOfObjects / @distinctUnionOfObjects

```
NSArray *inventory = @[iPhone5, iPhone5, iPhone5, iPadMini, macBookPro, macBookPro];
[inventory valueForKeyPath:@"@unionOfObjects.name"];

[inventory valueForKeyPath:@"@distinctUnionOfObjects.name"];
// "iPhone 5", "iPad mini", "MacBook Pro"

```


### Array and Set Operators
1. @distinctUnionOfArrays / @unionOfArrays
2. @distinctUnionOfSets / 

```
telecomStoreInventory = @[iPhone5, iPhone5, iPadMini];

[@[appleStoreInventory, telecomStoreInventory] valueForKeyPath:@"@distinctUnionOfArrays.name"];
```
KVC Collection Operators are a must-know for anyone wanting to save a few extra lines of code and look cool in the process.