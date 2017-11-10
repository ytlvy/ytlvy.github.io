---
categories:
- IOS
date: '2015-08-22'
tags:
- C
title: C programming -- Array & String
---

[reference](http://www.cnblogs.com/kenshincui/p/3843505.html)

## C语言之数组和字符串

### 概览 

数组在C语言中有着特殊的地位，它有很多特性，例如它的存储是连续的，数组的名称就是数组的地址等。而在C语言中是没有String类型的，那么如果要表示一个字符串，就必须使用字符数组。今天主要就介绍如下三个方面：

- 一维数组
- 多维数组
- 字符串

### 一维数组

一维数组操作比较简单，但是需要注意，数组长度必须是固定的，长度不能使用变量进行初始化；如果声明的同时进行赋值则数组长度可以省略，编译器会自动计算数组长度；同时数组不能先声明再一次性赋值（当然可以对每个元素一一赋值）。

```
#include <stdio.h>

int main(){
    int len = 2;
    //int a[len] = { 1, 2};//错误,不能使变量
    int a[2];//正确
    a[0] = 1;
    a[1] = 2;
    //a[2] = 3;//超过数组长度，但是编译器并不会检查，运行报错
    int b['a'] = {1,2,3};//'a'=97，所以可以作为数组长度，但是后面的元素没有初始化，其值默认为0
    for (int i = 0; i < 97; ++i){
        printf("b[%d]=%d\n",i,b[i]);
    }
    int c[2 * 3];//2*3是固定值可以作为数组长度
    int d[] = { 1, 2, 3 };//如果初始化的同时赋值则数组长度可以省略，当前个数为3
}
```

<!--more-->

### 扩展--数组的存储

数组在内存中存储在一块连续的空间中，如果知道数组类型（int、float等）和初始地址就可以知道其他元素的地址，同时由于数组名等于数组第一个元素的地址，所以当数组作为参数（作为参数时形参可以省略）其实是引用传递。

```
#include <stdio.h>

int main(){
    int const l = 3;
    int a[l] = { 1, 2,3 };
    for (int i = 0; i < l; ++i){
        //由于当前在32位编译器下，int型长度为4个字节，可以判断出三个地址两两相差都是4
        printf("a[%d]=%d,address=%x\n", i, a[i], &a[i]);
    }
    /*当前输出结果：
    a[0] = 1, address = c9f95c
    a[1] = 2, address = c9f960
    a[2] = 3, address = c9f964*/
}
```

我们看一下上面定义的数组在内存中存储结构

![](http://images.cnitblog.com/blog/62046/201407/142058350217075.png)

再来看一下数组作为参数传递的情况，数组作为参数传递的是数组的地址
```
#include <stdio.h>

void changeValue(int a[]){ a[0] = 10;
}

int main(){ int a[2] = {1,2};
    changeValue(a); for (int i = 0; i < 2; ++i){
        printf("a[%d]=%d\n",i,a[i]);
    } /*打印结果
    a[0]=10
    a[1]=2
    */
}
```

### 多维数组

多维数组其实可以看成是一个特殊的一维数组，只是每个元素又是一个一维数组，下面简单看一下多维数组的初始化和赋值

```
#include <stdio.h>

int main(){
    int a[2][3];//2行3列，二维数组可以看成是一个特殊的一维数组，只是它的每一个元素又是一个一维数组
    a[0][0] = 1;
    a[0][1] = 2;
    a[0][2] = 3;
    a[1][0] = 4;
    a[1][1] = 5;
    a[1][2] = 6;
    for (int i = 0; i < 2; ++i){
        for (int j = 0; j < 3; ++j){
            printf("a[%d][%d]=%d,address=%x\n", i, j, a[i][j], &a[i][j]);
        }
    }
    /*打印结果
    a[0][0]=1,address=f8fb24
    a[0][1]=2,address=f8fb28
    a[0][2]=3,address=f8fb2c
    a[1][0]=4,address=f8fb30
    a[1][1]=5,address=f8fb34
    a[1][2]=6,address=f8fb38
    */
    //初始化并直接赋值
    int b[2][3] = { { 1, 2, 3 }, { 4, 5, 6 } };
    //由于数组的赋值顺序是先从第一行第一列，再第一行第二列...然后第二行第一列...，所以我们也可以写成如下形式
    int c[2][3] = { 1, 2, 3, 4, 5, 6 };
    //也可以只初始化部分数据，其余元素默认为0
    int d[2][3] = { 1, 2, 3, 4 };
    for (int i = 0; i < 2; ++i){
        for (int j = 0; j < 3; ++j){
            printf("d[%d][%d]=%d\n", i, j, d[i][j]);
        }
    }
    /*打印结果
    d[0][0]=1
    d[0][1]=2
    d[0][2]=3
    d[1][0]=4
    d[1][1]=0
    d[1][2]=0
    */
    //当然下面赋值也可以
    int e[2][3] = { {}, { 4, 5, 6 } };
    //可以省略行号,但是绝对不可以省略列号，因为按照上面说的赋值顺序，它无法判断有多少行
    int f[][3] = { {1,2,3},{4,5,6} };
}
```

#### 扩展--多维数组的存储

以上面a数组为例，它在内存中的结构如下图

![](http://images.cnitblog.com/blog/62046/201407/142058356937703.png)

根据上图和一维数组的存储，对于二维数组可以得出如下结论:数组名就是整个二维数组的地址，也等于第一行数组名的地址，还等于第一个元素的地址；第二行数组名等于第二行第一个元素的地址。用表达式表示：

a=a[0]=&a[0][0]
a[1]=&a[1][0]

同样可以得出a[i][j]=a[i]+j。关于三维数组、四维数组等多维数组，其实可以以此类推，在此不再赘述。

### 字符串

在C语言中是没有字符串类型的，如果要表示字符串需要使用char类型的数组，因为字符串本身就是多个字符的组合。但是需要注意的是字符串是一个特殊的数组，在它的结束位置必须要加一个”\0”（ASCII中0是空操作符，表示什么也不做）来表示字符串结束，否则编译器是不知道什么时候字符串已经结束的。当直接使用字符串赋值的时候程序会自动加上”\0”作为结束符。

```
//
//  main.c
//  ArrayAndString
//
//  Created by KenshinCui on 14-7-06.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#include <stdio.h>

int main(int argc, const char * argv[])
{

    char a[] = {'K','e','n','s','h','i','n','\0'};
    printf("%s",a); //结果：Kenshin，注意使用%s输出字符串内容，如果换成整形输出格式其实输出的是a的地址
    printf("\n");
    printf("address=%x", a); //结果：address=5fbff890
    printf("\n");
    //后面的\0绝对不能省略,如果没有\0则会出现如下情况
    char b[] = { 'I', 'a', 'm'};
    printf("%s",b); //没有按照期望输出，多了一些垃圾数据，在当前环境打印结果：IamKenshin
    printf("\n");
    printf("address=%x",b); //结果：address=5fbff88d
    printf("\n");
    //直接赋值为字符串，此时不需要手动添加\0，编译器会自动添加
    char c[] = "Kenshin";
    printf("c=%s",c); //结果：c=Kenshin
    printf("\n");
    
    //二维数组存储多个字符串
    char d[2][3]={"Kenshin","Kaoru","Rose","Jack","Tom","Jerry"};
    
    
    return 0;
}
```

从上面代码注释中可以看到打印b的时候不是直接打印出来“Iam”而是打印出了“IamKenshin”，原因就是编译器无法判断字符串是否结束，要解释为什么打印出“IamKenshin”我们需要了解a和b在内存中的存储。

![](http://images.cnitblog.com/blog/62046/201407/142058363658331.png)

从图中我们不难发现由于a占用8个字节，而定义完a后直接定义了b，此时分配的空间连续，b占用3个字节，这样当输出b的时候由于输出完“Iam”之后并未遇到”\0”标记，程序继续输出直到遇到数组a中的“\0”才结束，因此输出内容为“IamKenshin”。

#### 扩展--字符串操作常用函数

下面简单看一下和字符和字符串相关的常用的几个函数

```
//
//  main.c
//  ArrayAndString
//
//  Created by Kenshin Cui on 14-7-04.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#include <stdio.h>

int main(int argc, const char * argv[])
{
    /*字符操作*/
    putchar('a'); //结果：a，putchar一次只能输出一个字符
    printf("\n");
    putchar(97);//结果:a
    printf("\n");
    char a;
    a=getchar();//getchar()一次只能接收一个字符，可以接收空格、tab、回车
    printf("a=%c",a);
    printf("\n");

    /*字符串操作*/
    char b[]="Kenshin";
    printf("b=%s",b);
    printf("\n");
    puts(b); //puts用于输出单个字符串，不能像printf格式化输出，会自动添加换行
    printf("\n");
    
    char c[10];
    scanf("%s",c);//注意c没必要写成&c，因为c本身就代表了数组的地址
    printf("c=%s\n",c);//注意即使你输入的内容大于10，也能正确输出，但是下面的gets()函数却不行
    printf("\n");
    
    //gets()函数，注意它是不安全的，因为接收的时候不知道它的大小容易造成溢出，建议不要使用
    char d[10];
    gets(d); //gets一次只能接收一个字符串，但是scanf可接收多个；scanf不能接收空格、tab，gets则可以
    printf("d=%s",d);
    printf("\n");
    
    char e[]={'K','s','\0'};
    printf("%lu",strlen(e)); //结果是：2，不是3，因为\0不计入长度
    printf("\n");
    char f[]={"Kenshin"};
    printf("%lu",strlen(f)); //结果是：7
    printf("\n");
    
    char g[5];
    strcpy(g,"hello,world!");
    printf("%s",g); //结果是：hello,即使定义的g长度为5，但是也能完全拷贝进去
    printf("\n");
    char h[5];
    char i[]={'a','b','c','\0','d','e','f','\0'};
    strcpy(h,i);
    printf("%s",h); //结果是：abc,遇到第一个\0则结束
    printf("\n");
    
    strcat(i,"ghi");
    printf("%s",i); //结果是：abcghi,注意不是abcdefghi,strcat，从i第一\0开始使用“ghi”覆盖，覆盖完之后加上一个\0,在内存中目前应该是：{'a','b','c','g','h','i','\0','f','\0'}
    printf("\n");
    
    char j[]="abc";
    char k[]="aBc";
    char l[]="acb";
    char m[]={'a','\0'};
    printf("%d,%d,%d",strcmp(j,k),strcmp(k,l),strcmp(l,m));//遇到第一个不相同的字符或\0则返回两者前后之差，结果：32,-33,99
    printf("\n");

    return 0;
}
```

> 1.在Xcode中会提示gets是不安全的，推荐使用fgets()。
2.strlen()只用于计算字符串长度，由于在C语言中字符串使用字符数组长度表示，所以它可以计算带有’\0’结尾的字符数组长度，但是它并不能计算其他类型的数组长度。




