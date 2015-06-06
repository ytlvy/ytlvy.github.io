---
layout: post
title: "ios CFStringTransform"
date: 2015-06-05 09:31:29 +0800
comments: true
categories: [IOS, Swift]
---

```
var mutableString = NSMutableString(string: "Hello! こんにちは! สวัสดี! مرحبا! 您好!") as CFMutableStringRef
CFStringTransform(mutableString, nil, kCFStringTransformToLatin, Boolean(0))
//Hello! こんにちは! สวัสดี! مرحبا! 您好! → Hello! kon'nichiha! s̄wạs̄dī! mrḥbạ! nín hǎo!

CFStringTransform(mutableString, nil, kCFStringTransformStripCombiningMarks, Boolean(0))
//Hello! kon'nichiha! s̄wạs̄dī! mrḥbạ! nín hǎo! → Hello! kon'nichiha! swasdi! mrhba! nin hao!

let tokenizer = CFStringTokenizerCreate(nil, mutableString, CFRangeMake(0, CFStringGetLength(mutableString)), 0, CFLocaleCopyCurrent())

var mutableTokens: [String] = []
var type: CFStringTokenizerTokenType
do {
    type = CFStringTokenizerAdvanceToNextToken(tokenizer)
    let range = CFStringTokenizerGetCurrentTokenRange(tokenizer)
    let token = CFStringCreateWithSubstring(nil, mutableString, range) as NSString
    mutableTokens.append(token)
} while type != .None

//(hello, kon'nichiha, swasdi, mrhba, nin, hao)
```
