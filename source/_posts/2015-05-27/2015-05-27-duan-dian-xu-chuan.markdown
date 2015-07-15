---
layout: post
title: "断点续传"
date: 2015-05-27 15:46:45 +0800
comments: true
categories: IOS
---
    [request addValue:@"bytes=500-" forHTTPHeaderField:@"Range"];
    _fileHandle = [NSFileHandle fileHandleForWritingAtPath:_tempFilePath];
    _fileOffset = _fileHandle ? [_fileHandle seekToEndOfFile] : 0;