---
layout: post
title: "IOS NSSortDescriptor"
date: 2015-06-06 20:54:35 +0800
comments: true
categories: IOS
tags: [ios, sort]
---

## NSSortDescriptor
NSSortDescriptor objects are constructed with the following parameters:
1. key: for a given collection, the key for the corresponding value to be sorted on for each object in the collection
2. ascending: a boolean specifying whether the collection should be sorted in ascending (YES) or descending (NO) order
3. a selector (SEL) or comparator (NSComparator)

<!--more-->

```
NSArray *firstNames = @[@"Alice", @"Bob", @"Charlie", @"Quentin"];
NSArray *lastNames = @[@"Smith", @"Jones", @"Smith", @"Alberts"];
NSArray *ages = @[@24, @27, @33, @31];
NSMutableArray *people = [NSMutableArray array];

[firstNames enumerateObjectsUsingBlock: ^(id obj, NSUInteger idx, BOOL *stop) {
    Person *person = [[Person alloc] init];
    person.firstName = [firstNames objectAtIndex:idx];
    person.lastName = [lastNames objectAtIndex:idx];
    person.age = [ages objectAtIndex:idx];
    [people addObject:person];
}];

NSSortDescriptor *firstNameSortDescriptor = 
    [NSSortDescriptor sortDescriptorWithKey:@"firstName"
                                        ascending: YES
                                        selector:@selector(localizedStandardCompare:)];
NSSortDescriptor *lastNameSortDescriptor =
    [NSSortDescriptor sortDescriptorWithKey:@"lastName"
                                    ascending: YES
                                    selector: @selector(localizedStandardCompare:)];
[people sortedArrayUsingDescriptor:@[firstNameSortDescriptor, lastNameSortDescriptor]];

```
