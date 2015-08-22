title: C programming -- Struct
categories: IOS
tags:
  - C
date: 2015-08-22 20:45:36
---
@charset "UTF-8";
/**
 * 
 * @authors Your Name (you@example.org)
 * @date    2015-08-16 11:06:45
 * @version $Id$
 * Copyright (c) 2015年 Guo yanjie. All rights reserved.
 */

[reference](http://www.cnblogs.com/kenshincui/p/3856543.html)

## C语言之构造类型

### 概述

在第一节中我们就提到C语言的构造类型，分为：数组、结构体、枚举、共用体，当然前面数组的内容已经说了很多了，这一节将会重点说一下其他三种类型。

- 结构体
- 枚举
- 共用体

### 结构体

数组中存储的是一系列相同的数据类型，那么如果想让一个变量存储不同的数据类型就要使用结构体，结构体定义类似于C++、C#、Java等高级语言中类的定义，但事实上它们又有着很大的区别。结构体是一种类型，并非一个变量，只是这种类型可以由其他C语言基本类型共同组成。

```
//
//  main.c
//  ConstructedType
//
//  Created by Kenshin Cui on 14-7-18.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#include <stdio.h>

//结构体类型Date
struct Date{
    int year;
    int month;
    int day;
};

struct Person{
    char *name;
    int age;
    struct Date birthday;//一个结构体中使用了另一个结构体类型，结构体类型变量声明前必须加上struct关键字
    float height;
};

int main(int argc, const char * argv[]) {
    struct Person p={"Kenshin",28,{1986,8,8},1.72};
    //定义结构体变量并初始化,不允许先定义再直接初始化，例如：struct Person p;p={"Kenshin",28,{1986,8,8},1.72};是错误的，但是可以分别赋值，例如p.name="Kenshin"
    
    printf("name=%s,age=%d,birthday=%d-%d-%d,height=%.2f\n",p.name,p.age,p.birthday.year,p.birthday.month,p.birthday.day,p.height); 
    //结果：name=Kenshin,age=28,birthday=1986-8-8,height=1.72，结构体的引用是通过"结构体变量.成员名称"(注意和结构体指针访问结构体成员变量区分，结构体指针使用p->a的形式访问)
    
    printf("len(Date)=%lu,len(Person)=%lu\n",sizeof(struct Date),sizeof(struct Person)); 
    //结果：len(Date)=12,len(Person)=32
    
    return 0;
}
```

<!-- more -->

对于上面的例子需要做出如下说明：

- 可以在定义结构体类型的同时声明结构体变量；
- 如果定义结构体类型的同时声明结构体变量，此时结构体名称可以省略；
- 定义结构体类型并不会分配内存，在定义结构体变量的时候才进行内存分配（同基本类型时类似的）；
- 结构体类型的所占用内存大型等于所有成员占用内存大小之和（如果不考虑内存对齐的前提下）；

对第4点需要进行说明，例如上面代码是在64位编译器下运行的结果（int长度4，char长度1，float类型4），Date=4+4+4=12。但是对于Person却没有那么简单了，因为按照正常方式计算Person=8+4+12+4=28，但是从上面代码中给出的结果是32，为什么呢？这里不得不引入一个概念“内存对齐”，关于内存对齐的概念在这里不做详细说明，大家需要了解的是：在Mac OS X中对齐参数默认为8（可以通过在代码中添加#pragma pack(8)改变对齐参数），如果结构体中的类型不大于8，那么结构体长度就是其成员类型之和，但是如果成员变量的长度大于这个对齐参数那么得到的结果就不一定是各个成员变量之和了。Person类型的长度之所以是32，其实主要原因是因为Date类型长度12在存储时其偏移量12不是8的倍数，考虑到内存对齐的原因需要添加4个补齐长度，这里使用表格的形式列出了具体原因：

![](http://images.cnitblog.com/blog/62046/201407/201858560376210.png)

接下来看一下结构体数组、指向结构体的指针：

```
//
//  main.c
//  ConstructedType
//
//  Created by Kenshin Cui on 14-7-18.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#include <stdio.h>

struct Date{
    int year;
    int month;
    int day;
};

struct Person{
    char *name;
    int age;
    struct Date birthday;
    float height;
};

void changeValue(struct Person person){
    person.height=1.80;
}

int main(int argc, const char * argv[]) {
    struct Person persons[]={
        {"Kenshin",28,{1986,8,8},1.72},
        {"Kaoru",27,{1987,8,8},1.60},
        {"Rosa",29,{1985,8,8},1.60}
    };
    for (int i=0; i<3; ++i) {
        printf("name=%s,age=%d,birthday=%d-%d-%d,height=%.2f\n",
               persons[i].name,
               persons[i].age,
               persons[i].birthday.year,
               persons[i].birthday.month,
               persons[i].birthday.day,
               persons[i].height);
    }
    /*输出结果：
     name=Kenshin,age=28,birthday=1986-8-8,height=1.72
     name=Kaoru,age=27,birthday=1987-8-8,height=1.60
     name=Rosa,age=29,birthday=1985-8-8,height=1.60
     */
    
    
    
    struct Person person=persons[0];
    changeValue(person);
    printf("name=%s,age=%d,birthday=%d-%d-%d,height=%.2f\n",
           persons[0].name,
           persons[0].age,
           persons[0].birthday.year,
           persons[0].birthday.month,
           persons[0].birthday.day,
           persons[0].height);
    /*输出结果：
     name=Kenshin,age=28,birthday=1986-8-8,height=1.72
     */
    
    
    struct Person *p=&person;
    printf("name=%s,age=%d,birthday=%d-%d-%d,height=%.2f\n",
           (*p).name,
           (*p).age,
           (*p).birthday.year,
           (*p).birthday.month,
           (*p).birthday.day,
           (*p).height);
    /*输出结果：
     name=Kenshin,age=28,birthday=1986-8-8,height=1.72
     */
    printf("name=%s,age=%d,birthday=%d-%d-%d,height=%.2f\n",
           p->name,
           p->age,
           p->birthday.year,
           p->birthday.month,
           p->birthday.day,
           p->height);
    /*输出结果：
     name=Kenshin,age=28,birthday=1986-8-8,height=1.72
     */
    
    return 0;
}
```

结构体作为函数参数传递的是成员的值（值传递而不是引用传递），对于结构体指针而言可以通过”->”操作符进行访问。

### 枚举

枚举类型是比较简单的一种数据类型，事实上在C语言中枚举类型是作为整形常量进行处理的，通常称为“枚举常量”。

```
//
//  main.c
//  ConstructedType
//
//  Created by Kenshin Cui on 14-7-18.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#include <stdio.h>

enum Season{ //默认情况下spring=0，summer=1,autumn=2,winter=3
    spring,
    summer,
    autumn,
    winter
};

int main(int argc, const char * argv[]) {
    enum Season season=summer; //枚举赋值,等价于season=1
    printf("summer=%d\n",season); //结果：summer=1
    
    for(season=spring;season<=winter;++season){
        printf("element value=%d\n",season);
    }
    /*结果：
     element value=0
     element value=1
     element value=2
     element value=3
     */
    return 0;
}
```

需要注意的是枚举成员默认值从0开始，如果给其中一个成员赋值，其它后面的成员将依次赋值，例如上面如果summer手动指定为8，则autumn=9，winter=10，而sprint还是0。

### 共用体

共用体又叫联合，因为它的关键字是union（貌似数据库操作经常使用这个关键字），它的使用不像枚举和结构体那么频繁，但是作为C语言中的一种数据类型我们也有必要弄清它的用法。从前面的分析我们知道结构体的总长度等于所有成员的和（当然此时还可能遇到对齐问题），但是和结构体不同的是共用体所有成员共用一块内存，顺序从低地址开始存放，一次只能使用其中一个成员，union最终大小由共用体中最大的成员决定，对某一成员赋值可能会覆盖另一个成员。

```
//
//  main.c
//  ConstructedType
//
//  Created by Kenshin Cui on 14-7-20.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#include <stdio.h>

union Type{
    char a;
    short int b;
    int c;
};

int main(int argc, const char * argv[]) {
    union Type t;
    t.a='a';
    t.b=10;
    t.c=65796;
    
    printf("address(Type)=%x,address(t.a)=%x,address(t.b)=%x,address(t.c)=%x\n",&t,&t.a,&t.b,&t.c);
    //结果：address(Type)=5fbff7b8,address(t.a)=5fbff7b8,address(t.b)=5fbff7b8,address(t.c)=5fbff7b8
    
    printf("len(Type)=%d\n",sizeof(union Type));
    //结果：len(Type)=4
    
    printf("t.a=%d,t.b=%d,t.c=%d\n",t.a,t.b,t.c);
    //结果:t.a=4,t.b=260,t.c=65796
    
    return 0;
}
```

这里需要重点解释一个问题：为什么t.a、t.b、t.c输出结果分别是4、260、65796，当然t.c等于65796并不奇怪，但是t.a前面赋值为’a’不应该是97吗，而t.b不应该是10吗？其实如果弄清这个问题共用体的概念基本就清楚了。

根据前面提到的，共用体其实每次只能使用其中一个成员，对于上面的代码经过三次赋值最终使用的其实就是t.c,而通过上面的输出结果我们也确实看到c是有效的。共用体有一个特点就是它的成员存储在同一块内存区域，这块区域的大小需要根据它的成员中长度最大的成员长度而定。由于上面的代码是在64位编译器下编译的，具体长度：char=1，short int=2，int=4，所以得出结论，Type的长度为4，又根据上面输出的地址，可以得到下面的存储信息(注意数据的存储方式：高地址存储高位，低地址存储地位)：

![](http://images.cnitblog.com/blog/62046/201407/201858575218722.png)

```
当读取c的时候，它的二进制是“00000000  00000001  00000001  00000100”，换算成十进制就是65796；而经过三次赋值后，此时b的存储就已经被c成员的低位数据覆盖，b的长度是二，所以从起始地址取两个字节得到的二进制数据此时是“00000001  00000100”（b原来的数据已经被c低2位数据覆盖，其实此时就是c的低2位数据），换算成十进制就是260；类似的a此时的数据就是c的低一位数据”00000100”,换算成十进制就是4。
```




































