---
layout: post
title: "ios 文件处理"
date: 2015-05-27 15:45:51 +0800
comments: true
categories: IOS
---
#### path
```
    NSMutableArray *marray=[NSMutableArray array];
    NSString *path=[NSString pathWithComponents:marray];

    [path pathComponents]
    [path isAbsolutePath]
    NSLog(@"%@",[path lastPathComponent]);//取得最后一个目录
    [path stringByDeletingLastPathComponent]
    [path stringByAppendingPathComponent:@"Documents"]
    [path pathExtension]
    [path stringByDeletingPathExtension]
    [@"Users/KenshinCui/Desktop/test" stringByAppendingPathExtension:@"mp3"]
```

<!--more-->

#### NSFileManager
```
    NSFileManager *manager=[NSFileManager defaultManager];
    BOOL result= [manager createDirectoryAtPath:myPath withIntermediateDirectories:YES attributes:nil error:nil];
    [manager moveItemAtPath:myPath toPath:newPath error:&error]
    [manager changeCurrentDirectoryPath:newPath]

    //遍历整个目录
    NSString *path;
    NSDirectoryEnumerator *directoryEnumerator= [manager enumeratorAtPath:newPath];
    while (path=[directoryEnumerator nextObject]) {
        NSLog(@"%@",path);
    }

    //或者这样遍历
    NSArray *paths= [manager contentsOfDirectoryAtPath:newPath error:nil];
    NSObject *p;
    for (p in paths) {
        NSLog(@"%@",p);
    }

    [manager fileExistsAtPath:filePath isDirectory:NO]
    [manager isReadableFileAtPath:filePath]
    [manager contentsEqualAtPath:filePath andPath:filePath2]
    [manager moveItemAtPath:filePath toPath:newPath error:nil]
    [manager copyItemAtPath:newPath toPath:filePath3 error:nil]
    NSDictionary *attributes=[manager attributesOfItemAtPath:newPath error:nil]
    NSData *data=[manager contentsAtPath:filePath]
    [[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding]
    NSData *data2=[str2 dataUsingEncoding:NSUTF8StringEncoding]
```

#### NSFileHandle
```
    NSFileHandle *fileHandle=[NSFileHandle fileHandleForReadingAtPath:filePath];
    NSFileHandle *fileHandle2=[NSFileHandle fileHandleForWritingAtPath:newPath];
    [fileHandle2 writeData:data];
    [fileHandle2 closeFile];

    [fileHandle seekToFileOffset:12];
    NSData *data2= [fileHandle readDataToEndOfFile];
    [fileHandle seekToFileOffset:6];
    NSData *data3=[fileHandle readDataOfLength:5];
    [fileHandle closeFile];

    NSString *path=NSTemporaryDirectory();
```

#### NSBundle
```
    //利用NSBundle在程序包所在目录查找对应的文件
    NSBundle *bundle=[NSBundle mainBundle];//主要操作程序包所在目录
    //如果有test.txt则返回路径，否则返回nil
    NSString *path=[bundle pathForResource:@"test" ofType:@"txt"];//也可以写成：[bundle pathForResource:@"instructions.txt" ofType:nil];
    NSLog(@"%@",path);
    //结果：/Users/kenshincui/Library/Developer/Xcode/DerivedData/FoundationFramework-awxjohcpgsqcpsanqofqogwbqgbx/Build/Products/Debug/test.txt
    NSLog(@"%@",[NSString stringWithContentsOfFile:path encoding:NSUTF8StringEncoding error:nil]);

    //假设我们在程序运行创建一个Resources目录，并且其中新建pic.jpg，那么用下面的方法获得这个文件完整路径
    NSString *path1= [bundle pathForResource:@"pic" ofType:@"jpg" inDirectory:@"Resources"];
    NSLog(@"%@",path1);
```
