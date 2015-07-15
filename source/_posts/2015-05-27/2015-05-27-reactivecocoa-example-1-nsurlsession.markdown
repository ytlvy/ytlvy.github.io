---
layout: post
title: "ReactiveCocoa应用1---NSURLSession封装"
date: 2015-05-27 15:51:33 +0800
comments: true
categories: [IOS, FP]
---

### ReactiveCocoa应用1---NSURLSession封装
```
+ (RACSignal *)fetchJSONFromSession:(NSURLSession *)session withRequest:(NSURLRequest *)request {
    return [[[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber)
    {
        NSURLSessionDataTask *dataTask = [session dataTaskWithRequest:request completionHandler:^(NSData *data, NSURLResponse *response, NSError *error)
        {
            if (! error) {
                NSError *jsonError = nil;
                NSString* newStr = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
                id json = [NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:&jsonError];
                if (! jsonError) {
                    NSDictionary *dataOfDictionary = (NSDictionary *)json;
                    if (dataOfDictionary[@"stat"] && [dataOfDictionary[@"stat"] integerValue] == 0)
                    {
                        [subscriber sendNext: json[@"data"]];
                        [subscriber sendCompleted];
                    }else{
                        NSString *errmsg = dataOfDictionary[@"msg"];
                        if (![errmsg isKindOfClass:[NSString class]]) {
                            errmsg = @"服务器未知错误";
                        }
                       [subscriber sendError:[NSError errorWithDomain:[AppConstants httpHost] code:[dataOfDictionary[@"stat"] integerValue] userInfo:@{NSLocalizedDescriptionKey:errmsg}]];
                    }
                }
                else {
                    DDLogError(@"function: json 组装错误:%@", [jsonError localizedDescription]);
                    if (jsonError != nil) {
                        [subscriber sendError:jsonError];
                    }
                }
            }
            else {
                DDLogError(@"function: dataTask error: %@", [error localizedDescription]);
                [subscriber sendError:error];
            }
            
        }];
        
        [dataTask resume];
        
        return [RACDisposable disposableWithBlock:^{
            [dataTask cancel];
        }];
    }] logError] replayLazily];
}

```