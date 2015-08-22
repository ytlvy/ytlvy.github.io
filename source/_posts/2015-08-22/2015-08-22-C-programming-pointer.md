title: C programming -- pointer
categories: IOS
tags:
  - C
date: 2015-08-22 20:43:37
---
[reference](http://www.cnblogs.com/kenshincui/p/3848442.html)

## C语言之指针

### 概览

指针是C语言的精髓，但是很多初学者往往对于指针的概念并不深刻，以至于学完之后随着时间的推移越来越模糊，感觉指针难以掌握，本文通过简单的例子试图将指针解释清楚，今天的重点有几个方面：

- 什么是指针
- 数组和指针
- 函数指针

### 什么是指针

存放变量地址的变量我们称之为“指针变量”,简单的说变量p中存储的是变量a的地址,那么p就可以称为是指针变量,或者说p指向a。当我们访问a变量的时候其实是程序先根据a取得a对应的地址，再到这个地址对应的存储空间中拿到a的值，这种方式我们称之为“直接引用”；而当我们通过p取得a的时候首先要先根据p转换成p对应的存储地址，再根据这个地址到其对应的存储空间中拿到存储内容，它的内容其实就是a的地址，然后根据这个地址到对应的存储空间中取得对应的内容，这个内容就是a的值，这种通过p找到a对应地址再取值的方式成为“间接引用”。这里以表格形式列出a和p的存储以帮助大家理解上面说的内容：

![](http://images.cnitblog.com/blog/62046/201407/161307139741921.png)

<!-- more -->

接下来，看一下指针的赋值

```
//
//  main.c
//  Point
//
//  Created by Kenshin Cui on 14-7-05.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#include <stdio.h>

int main(int argc, const char * argv[]) {
    
    int a=1;
    int *p;
    p=&a; //也可以直接给指针变量赋值：int *p=&a;
    printf("address(a)=%x,address(p)=%x\n",&a,p); //结果：address(a)=5fbff81c,address(p)=5fbff81c
    printf("a=%d,p=%d\n",a,*p); //结果：a=1,p=1
    *p=2;
    printf("a=%d,*p=%d\n",a,*p); //结果：a=2,p=2
    
    int b=8;
    char c= 1;
    int *q=&c;
    printf("address(b)=%x,address(c)=%x\n",&b,&c);//结果：
    printf("c=%d,q=%d\n", c, *q); //结果：c=1,q=2049，为什么q的值不是1呢？
    
    return 0;
}
```

需要说明两点：

a. int *p;中的*只是表示p变量是一个指针变量；而打印*p的时候，*p中的*是操作符，表示p指针指向的变量的存储空间（当前存储就是1），同时我们也看到了*p==a；修改了*p也就是修改了p指向的存储空间的内容，也就修改了a，所以第二次打印a=2;

b. 指针所指向的类型必须和定义指针时声明的类型相同；上面指针q定义成了int型而指向了char型，结果输出*q打印出了2049，具体原因见下图（假设在16位编译器下，指针长度为2字节）

![](http://images.cnitblog.com/blog/62046/201407/161307173968575.png)

由于局部变量是存储在栈里面的，所以先存储b再存储a、p，当打印*p的时候，其实就是以p指向的地址对应的空间开始取两个字节的数据（因为定义p的时候它指向的是int型，在16位编译器下int类型的长度为2），刚好定义的b和c空间连续，所以就取到b的其中一个字节，最后*p二进制存储为“0000100000000001”（见上图黄色背景内容），十进制表示就是2049；

c. 指针变量占用的空间和它所指向的变量类型无关，只跟编译器位数有关（准确的说只跟寻址方式有关）；

### 数组和指针

由于数组的存储是连续的，数组名就是数组的地址，这样一来数组和指针就有着很微妙的关系，先看以下例子：

```
//
//  main.c
//  Point
//
//  Created by Kenshin Cui on 14-7-05.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#include <stdio.h>

void changeValue(int a[]){
    a[0]=2;
}
void changeValue2(int *p){
    p[0]=3;
}

int main(int argc, const char * argv[]) {
    int a[]={1,2,3};
    int *p=&a[0]; //等价于：*p=a;
    
    printf("len=%lu\n",sizeof(int));//取得int长度为2
    
    //指针加1代表地址向后挪动所指向类型的长度位（这里类型是int，长度为2）
    //也就是说p指向a[0],p+1指向a[1]，以此类推，所以我们通过指针也可以取出数组元素
    for(int i=0;i<3;++i){
        //printf("a[%d]=%d\n",i,a[i]);
        printf("a[%d]=%d\n",i,*(p+i));//由于a就代表数组的地址，其实这里还可以写成*(a+i),但是注意这里*(p+i)可以写成*(p++),但是*(a+i)不能写成*(a++),因为数组名是常量
    }
    /*输出结果：
     a[0]=1
     a[1]=2
     a[2]=3
     */
     
    
    changeValue(p); //等价于：changeValue(a)
    for(int i=0;i<3;++i){
        printf("a[%d]=%d\n",i,a[i]);
    }
    /*输出结果：
     a[0]=2
     a[1]=2
     a[2]=3
     */
    
    changeValue2(a); //等价于：changeValue2(p)
    for(int i=0;i<3;++i){
        printf("a[%d]=%d\n",i,a[i]);
    }
    /*输出结果：
     a[0]=3
     a[1]=2
     a[2]=3
     */
    
    return 0;
}
```

从上面的例子我们可以得出如下结论：

- 数组名a==&a[0]==*p；
- 如果p指向一个数组，那么p+1指向数组的下一个元素，同时注意p+1移动的长度并不固定，具体需要根据p指向的数据类型而定；
- 指针可以写成p++形式，但是数组名不可以，因为数组名是常量
- 不管函数的形参为数组还是指针，实参都可以使用数组名或指针；

#### 扩展--字符串和指针

由于在C语言中字符串就是字符数组，下面不妨看一下字符串和数组的关系：
```
//
//  main.c
//  Point
//
//  Created by Kenshin on 14-7-05.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#include <stdio.h>

int main(int argc, const char * argv[]) {
    char a[]="Kenshin";
    printf("%x,%s\n",a,a);//结果：5fbff820,Kenshin，同一个变量a是输出字符串还是输出地址，根据格式参数而定
    printf(a); //结果：Kenshin
    printf("\n");
    
    char b[]="Kenshin";
    char *p=b;
    printf("b=%s,p=%s\n",b,p);//结果：b=Kenshin,p=Kenshin
    
    //指针存储的是地址，而数组名存储的也是地址，既然字符数组可以表示字符串，那么指向字符的指针同样也可以，如下方式可以更简单的定义一个字符串
    char *c="Kenshin"; //等价于char c[]="Kenshin";
    printf("c=%s\n",c); //结果：c=Kenshin
    
    return 0;
}
```

以上代码中注释基本已经很清楚了，这里需要指出是为什么printf(a)能够直接输出字符串呢？

我们看一下printf()的定义:int     printf(const char * __restrict, ...) __printflike(1, 2);

其实printf的参数要求是指向字符类型的指针，而结合上面的例子和我们之前说的，如果函数形参是指针类型那么可以传入函数名，因此也就能正确输出字符串的内容了。类似的还有上一篇文章中说的strcat()、strcpy()等函数均是如此。

### 函数指针

在弄清函数指针的问题之前，我们不妨先来看一下返回指针类型数据的函数，毕竟指针类型也是C语言的数据类型，下面以一个字符串转换为大写字符的程序为例，在这个例子中不仅可以看到返回值为指针类型的函数同时还可以看到前面说到的指针移动操作：

```
//
//  main.c
//  Point
//
//  Created by Kenshin Cui on 14-06-28.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#include <stdio.h>

char * toUpper(char *a){
    char *b=a; //保留最初地址，因为后面的循环会改变字符串最初地址
    int len='a'-'A'; //大小写ASCII码差值相等
    while (*a!='\0') { //字符是否结束
        if(*a>'a'&&*a<'z'){//如果是小写字符
            *(a++) -= len; //*a表示数组对应的字符（-32变为小写），a++代表移动到下一个字符
        }
    }
       return b;
}

int main(int argc, const char * argv[]) {
    char a[]="hello";
    char *p=toUpper(a);
    printf("%s\n",p); //结果：HELLO
    return 0;
}
```

大家都是知道函数只能有一个返回值，如果需要返回多个值，怎么办，其实很简单，只要将指针作为函数参数传递就可以了，在下面的例子中我们再次看到指针作为参数进行传递。

```
//
//  main.c
//  Point
//
//  Created by Kenshin Cui on 14-6-28.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#include <stdio.h>

int operate(int a,int b,int *c){
    *c=a-b;
    return a+b;
}

int main(int argc, const char * argv[]) {
    int a=1,b=2,c,d;
    d=operate(a, b, &c);
    printf("a+b=%d,a-b=%d\n",d,c);//结果：a+b=3,a-b=-1
    return 0;
}
```

函数也是在内存中存储的，当然函数也有一个起始地址（事实上函数名就是函数的起始地址），这里同样需要弄清函数指针的关系。函数指针定义的形式：返回值类型 (*指针变量名)(形参1，形参2)，拿到函数指针其实我们就相当于拿到了这个函数，函数的操作都可以通过指针来完成，而且通过前面的例子可以看到指针作为C语言的数据类型，可以作为参数、作为返回值，那么当然函数指针同样可以作为函数的参数和返回值：

```
//
//  main.c
//  Point
//
//  Created by Kenshin Cui on 14-6-28.
//  Copyright (c) 2014年 Kenshin Cui. All rights reserved.
//

#include <stdio.h>

int sum(int a,int b){
    return a+b;
}

int sub(int a,int b){
    return a-b;
}

//函数指针作为参数进行传递
int operate(int a,int b,int (*p)(int,int)){
    return p(a,b);
}

int main(int argc, const char * argv[]) {
    int a=1,b=2;
    int (*p)(int ,int)=sum;//函数名就是函数首地址,等价于：int (*p)(int,int);p=sum;
    int c=p(a,b);
    printf("a+b=%d\n",c); //结果：a+b=3
    
    
    //函数作为参数传递
    printf("%d\n",operate(a, b, sum)); //结果：3
    printf("%d\n",operate(a, b, sub)); //结果：-1
    
    return 0;
}
```

函数指针可以作为函数参数进行传递，实在太强大了，是不是想起了C#中的委托？记得C#书籍中经常提到委托类似于函数指针，其实说的就是上面的情况。需要注意的是，普通的指针可以写成p++进行移动，而函数指针写成p++并没有意义。



