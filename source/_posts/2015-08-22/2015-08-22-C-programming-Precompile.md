title: C programming -- Precompile
categories: IOS
tags:
  - C
date: 2015-08-22 20:46:56
---
[reference](http://www.cnblogs.com/kenshincui/p/3854242.html)

## C语言之预处理

### 概述

大家都知道一个C程序的运行包括编译和链接两个阶段，其实在编译之前预处理器首先要进行预处理操作，将处理完产生的一个新的源文件进行编译。由于预处理指令是在编译之前就进行了，因此很多时候它要比在程序运行时进行操作效率高。在C语言中包括三类预处理指令，今天将一一介绍：

- 宏定义
- 条件编译
- 文件包含

### 宏定义

对于程序中经常用到的一些常量或者简短的函数我们通常使用宏定义来处理，这样做的好处是对于程序中所有的配置我们可以统一在宏定义中进行管理，而且由于宏定义是在程序编译之前进行替换相比定义成全局变量或函数效率更高。

```
//
//  main.c
//  Pretreatment
//
//  Created by Kenshin Cui on 14-6-28.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#include <stdio.h>
#define PI 3.14 //宏定义一般大写
#define R 10
#define S 2*PI*R //在另一个宏里面引用了上面的宏

int main(int argc, const char * argv[]) {
    float r=10.5;
    double area=PI*r*r;
    printf("area=%.2f\n",area);
    
    double a=S;
    printf("a=%.2f\n",a);
    printf("PI=3.14\n");//注意输出结果不是3.14=3.14而是PI=3.14，字符串中的PI并不会被替换
#undef PI //强制终止宏定义，否则它的范围一直到文件结束
    int PI=3.1415926;
    double area2=PI*r*r;
    printf("area2=%.2f\n",area2);
    
    
    return 0;
}
```

宏定义实际的操作就是在预处理时进行对应替换，这个阶段不管语法是否正确，而且对于字符串中出现的宏名不会进行替换。宏定义的功能事实上是非常强大的，除了简单的常量替换还可以传入参数：

<!-- more -->

```
//
//  1.2.c
//  Pretreatment
//
//  Created by Kenshin Cui on 14-7-17.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#include <stdio.h>
#define SUM(a,b) a+b
#define SUB(a,b) (a-b)
#define MUL (a,b) (a*b) //这么定义是错误的，预处理器会认为宏名为”MUL“,替换内容为”(a,b) (a*b)“


int main(int argc, const char * argv[]) {
    
    int a=2,b=3,c,d;
    c=SUM(a, b);
    printf("c=%d\n",c); //结果：c=5
    d=SUM(a, b)*2;
    printf("d=%d\n"); //结果：8,为什么不是10呢？因为替换后：d=a+b*2也就是2+3*2=8
    
    int e=SUB(b, a)*2;
    printf("(b-a)*2=%d\n",e); //结果：2,如果SUB定义时不加括号这里应该是-1
    
    return 0;
}
```

上面我们可以看出带参数的宏功能很强大，有点类似于函数，同函数不同的是它只是简单的替换，不涉及存储空间分配，参数、返回值等问题，但是由于它在预处理阶段展开，所以一般效率较高。使用带参数的宏需要注意的就是结果最好用括号括起来否则很容易出现问题（在上面的SUM例子中我们应该已经看到了）；还有一点就是带参数的宏定义时名称和参数之间不要有空格。

### 条件编译

条件编译其实就是在编译之前预处理器根据预处理指令判断对应的条件，如果条件满足就将对应的代码编译进去，否则代码就根本不进入编译环节（相当于根本就没有这段代码）。

```
//
//  main.c
//  Pretreatment
//
//  Created by Kenshin Cui on 14-06-28.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#include <stdio.h>
#define COUNT 1

int main(int argc, const char * argv[]) {
    
//判断是否定义了 COUNT 宏
#if defined(COUNT) //等价于：#ifdef COUNT,相反如果判断没有定义过则可以通过#if !defined(COUNT)或者#ifndef COUNT
    printf("COUNT defined\n");
#endif
    
//判断宏定义COUNT是否等于1
#if COUNT==1
    showMessage("hello,world!\n");
#else
    say();
#endif
    
    return 0;
}
```

### 文件包含

文件包含指令#include在前面也多次使用过，这里再次强调一下。首先使用#include"xxx"包含和使用#include <xxx>包含的不同之处就是使用<>包含时，预处理器会搜索C函数库头文件路径下的文件，而使用""包含时首先搜索程序所在目录，其次搜索系统Path定义目录，如果还是找不到才会搜索C函数库头文件所在目录。

另外在使用#include的时候我们需要注意包含文件的时候是不能递归包含的，例如a.h文件包含b.h，而b.h就不能再包含a.h了；还有就是重复包含虽然是允许的（这里指的是重复包含头文件）但是这会降低编译性能，不妨看一下下面的例子：

![](http://images.cnitblog.com/blog/62046/201407/182035388189767.png)

上面有三段代码，在main.c和person.h中都包含了message.h而main.c自身又包含了person.h,这样程序在预处理阶段会对包含内容进行替换，替换后mian.c中包含了两个#include “message.h”虽然没有报错，但这会影响编译的性能，正确的做法应该是这样的：

```
#ifndef _PERSON_H_
#define _PERSON_H_

#include "person.h"

#endif
```

![](http://images.cnitblog.com/blog/62046/201407/182035402567293.png)

其实就是用宏定义判断一个宏是否定义了，如果没有定义则会定义这个宏，这样以来如果已经包含过则这个宏定义肯定已经定义过了，即使再包含也不会重新定义了，下面的代码也就不会包含进去。



