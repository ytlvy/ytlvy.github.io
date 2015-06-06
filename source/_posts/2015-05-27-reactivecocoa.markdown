---
layout: post
title: "Reactivecocoa"
date: 2015-05-27 15:55:44 +0800
comments: true
categories: [IOS, FP]
---
### Reactivecocoa
#### zip
    [[[[RACSignal zip:@[RACObserve(self, minimum), RACObserve(self, maximum),
        RACObserve(self, average)]] skip:1] reduceEach:^id{
            return nil;
        }] subscribeNext:^(id x) {
            [self buildView]; //called once, while all three values were changed.
    }];

#### RACTuple
    [[RACSignal  combineLatest:signals] map:^(RACTuple *strings) {
        for (NSString *string in strings) {
            // Do whatever here.
        }
        return nil;
    }];