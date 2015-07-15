
title: Stack And Heap OverFlow
categories: IOS
tags: 
  - Stack
  - Heap
  - OverFlow
description: 
date: 2015-07-03 09:44:56
author:
photos:
---
翻译自https://developer.apple.com/library/mac/documentation/Security/Conceptual/SecureCodingGuide/Articles/BufferOverflows.html

## 栈溢出
> 在大部分操作系统中, 每个程序拥有一个***栈***(多线程程序,每个线程拥有一个单独的栈), 栈用于局部变量的存储. 
> 栈被划分为单元, 此单元被成为帧(frame), 每个帧包含了某个函数的某一调用的全部数据.这些数据包含: 函数参数, 全部的临时变量,和 linkage 信息(就是调用函数的地址. 以便当前代码执行完毕后, 返回原位置继续执行). 通过配置编译参数,也可以包含下一帧的顶部地址.数据的布局以及排序,由操作系统决定,不同系统略有差异.
> 当函数被调用的时候, 一个新的栈帧被添加到栈的顶部. 当函数返回时,顶部的栈帧被移除.在执行过程中, 程序只能直接访问栈顶帧得数据(指针除外, 但是不建议如此设计).这种设计,方便了递归调用的实现, 因为每个子递归调用拥有独立的临时变量和参数备份.
> 下图介绍了栈的组织结构.(这只是示意图, 实际情况要根据 cpu设计来决定)

![](https://developer.apple.com/library/mac/documentation/Security/Conceptual/SecureCodingGuide/Art/stack.gif)
<!-- more -->
![Memory Layout of C Programs](http://d2o58evtke57tz.cloudfront.net/wp-content/uploads/Memory-Layout-300x255.gif)

> 通常情况下, 程序应该检测所有的输入数据, 来确认此数据是否符合需求.不幸的是, 很多情况下, 程序员都假定用户输入是没有危害的.
> 当程序需要将数据存入固定大小的 buffer 时, 将发生灾难性后果. 恶意的攻击者可能提供大于存储区大小的数据, 因为函数只是在栈帧中分配了有限的空间, 这个数据会覆盖栈帧中得其他数据.
> 下面的例子, 演示了一个聪明的攻击者,可以通过这个技术来覆盖函数的返回值区域,来指向他自己的代码块.然后当此函数执行完毕,将不会按照原设计返回调用函数 funcB, 而是跳转到了攻击者的恶意代码.

![Figure 2-2  Stack after malicious buffer overflow](https://developer.apple.com/library/mac/documentation/Security/Conceptual/SecureCodingGuide/Art/cracked_stack.jpg)

除了攻击函数执行链( linkage )外, 攻击者可以通过更改临时变量的值,以及函数参数的方式来造成破坏.

### 个人理解
1. 由于栈是从高地址到低地址(high -> low)增长的
2. `main` 函数作为入口, 系统调用 main 函数的栈帧(只是部分数据, 因为系统函数的变量等已经入栈不再考虑)将第一个入栈.因此位于栈底,也就是说位于高地址位, 同时位于下图的顶部位置.
3. 此栈帧入栈顺序 `main 函数参数列表`(逆序入栈) ==> `main 函数返回地址` (被调用的下一条语句的地址)
4. `main` 函数栈帧 生成并入栈数据: `系统函数 EBP 地址`  ==> `main 函数临时变量列表`(正序入栈)
5. `main` 调用 `test` 入栈顺序: `test 参数列表`(逆序入栈,先入栈最后一个, 方便 EBP 计算获取) ==> `test 返回地址`
6. `test`函数栈帧 生成并入栈: `main 函数 EBP 地址` ==> `test 临时变量`(正序入栈)


#### 返回地址和EBP 入栈的意义
- 返回地址用来当前调用执行完毕后, 进行下一代码执行.
- EBP需要保留和恢复, 是因为 当前函数的三种主要数据需要依靠 EBP 来计算获得:
- ***临时变量***(EBP - 4..  向下 低地址) -4为第一个临时变量
- ***返回地址***(EBP + 4)
- ***参数列表***( EBP + 8 .. 向上高地址) +8 为第一个参数

所以当输入数据超过分配大小的时候, 数据会向高地址覆盖,而当前函数的执行的返回地址位于
![](/images/stackLayout.gif)

#### 函数调用栈
以下转自(http://hutaow.com/blog/2013/10/15/dump-stack/)

要了解调用栈，首先需要了解函数的调用过程，下面用一段代码作为例子：
```
#include <stdio.h>

int test(int a, int b) {
    int result = 0;

    result = a + b;

    return result;
}

int main(int argc, char *argv[]) {
    int result = 0;

    result = test(1, 2);

    printf("result = %d \r\n", result);

    return 0;
}
```

使用gcc编译，然后gdb反汇编main函数，看看它是如何调用add函数的：
```
(gdb) disassemble main 
Dump of assembler code for function main:
   0x08048439 <+0>:     push   %ebp
   0x0804843a <+1>:     mov    %esp,%ebp
   0x0804843c <+3>:     and    $0xfffffff0,%esp
   0x0804843f <+6>:     sub    $0x20,%esp       # 分配栈空间(栈向低地址方向生长)
   0x08048442 <+9>:     movl   $0x0,0x1c(%esp)  # 给result变量赋0值
   0x0804844a <+17>:    movl   $0x2,0x4(%esp)   # 将第2个参数压栈(该参数偏移为esp+0x04)
   0x08048452 <+25>:    movl   $0x1,(%esp)      # 将第1个参数压栈(该参数偏移为esp+0x00)
   0x08048459 <+32>:    call   0x804841c <add>  # 调用add函数
   0x0804845e <+37>:    mov    %eax,0x1c(%esp)  # 将add函数的返回值赋给result变量
   0x08048462 <+41>:    mov    0x1c(%esp),%eax
   0x08048466 <+45>:    mov    %eax,0x4(%esp)
   0x0804846a <+49>:    movl   $0x8048510,(%esp)
   0x08048471 <+56>:    call   0x80482f0 <printf@plt>
   0x08048476 <+61>:    mov    $0x0,%eax
   0x0804847b <+66>:    leave  
   0x0804847c <+67>:    ret    
End of assembler dump.
```
可以看到，参数是在add函数调用前压栈，换句话说，参数压栈由调用者进行，参数存储在调用者的栈空间中，下面再看一下进入add函数后都做了什么：
```
(gdb) disassemble add
Dump of assembler code for function add:
   0x0804841c <+0>:     push   %ebp             # 将ebp压栈(保存函数调用者的栈基址)
   0x0804841d <+1>:     mov    %esp,%ebp        # 将ebp指向栈顶esp(设置当前函数的栈基址)
   0x0804841f <+3>:     sub    $0x10,%esp       # 分配栈空间(栈向低地址方向生长)
   0x08048422 <+6>:     movl   $0x0,-0x4(%ebp)  # 给result变量赋0值(该变量偏移为ebp-0x04)
   0x08048429 <+13>:    mov    0xc(%ebp),%eax   # 将第2个参数的值赋给eax(准备运算)
   0x0804842c <+16>:    mov    0x8(%ebp),%edx   # 将第1个参数的值赋给edx(准备运算)
   0x0804842f <+19>:    add    %edx,%eax        # 加法运算(edx+eax)，结果保存在eax中
   0x08048431 <+21>:    mov    %eax,-0x4(%ebp)  # 将运算结果eax赋给result变量
   0x08048434 <+24>:    mov    -0x4(%ebp),%eax  # 将result变量的值赋给eax(eax将作为函数返回值)
   0x08048437 <+27>:    leave                   # 恢复函数调用者的栈基址(pop %ebp)
   0x08048438 <+28>:    ret                     # 返回(准备执行下条指令)
End of assembler dump.
```

进入add函数后，首先进行的操作是将当前的栈基址ebp压栈(此栈基址是调用者main函数的)，然后将ebp指向栈顶esp，接下来再进行函数内的处理流程。函数结束前，会将函数调用者的栈基址恢复，然后返回准备执行下一指令。这个过程中，栈上的空间会是下面的样子：
![](/images/dia_function_stack.png)

可以发现，每调用一次函数，都会对调用者的栈基址(ebp)进行压栈操作，并且由于栈基址是由当时栈顶指针(esp)而来，会发现，各层函数的栈基址很巧妙的构成了一个链，即当前的栈基址指向下一层函数栈基址所在的位置，如下图所示：
![](/images/dia_dump_stack.png)

了解了函数的调用过程，想要回溯调用栈也就很简单了，首先获取当前函数的栈基址(寄存器ebp)的值，然后获取该地址所指向的栈的值，该值也就是下层函数的栈基址，找到下层函数的栈基址后，重复刚才的动作，即可以将每一层函数的栈基址都找出来，这也就是我们所需要的调用栈了。

下面是根据原理实现的一段获取函数调用栈的代码，供参考。
```
#include <stdio.h>

/* 打印调用栈的最大深度 */
#define DUMP_STACK_DEPTH_MAX 16

/* 获取寄存器ebp的值 */
void get_ebp(unsigned long *ebp) {
    __asm__ __volatile__ (
        "mov %%ebp, %0"
        :"=m"(*ebp)
        ::"memory");
}

/* 获取调用栈 */
int dump_stack(void **stack, int size) {
    unsigned long ebp = 0;
    int depth = 0;

    /* 1.得到首层函数的栈基址 */
    get_ebp(&ebp);

    /* 2.逐层回溯栈基址 */
    for (depth = 0; (depth < size) && (0 != ebp) && (0 != *(unsigned long *)ebp) && (ebp != *(unsigned long *)ebp); ++depth) {
        stack[depth] = (void *)(*(unsigned long *)(ebp + sizeof(unsigned long)));
        ebp = *(unsigned long *)ebp;
    }

    return depth;
}

/* 测试函数 2 */
void test_meloner() {
    void *stack[DUMP_STACK_DEPTH_MAX] = {0};
    int stack_depth = 0;
    int i = 0;

    /* 获取调用栈 */
    stack_depth = dump_stack(stack, DUMP_STACK_DEPTH_MAX);

    /* 打印调用栈 */
    printf(" Stack Track: \r\n");
    for (i = 0; i < stack_depth; ++i) {
        printf(" [%d] %p \r\n", i, stack[i]);
    }

    return;
}

/* 测试函数 1 */
void test_hutaow() {
    test_meloner();
    return;
}

/* 主函数 */
int main(int argc, char *argv[]) {
    test_hutaow();
    return 0;
}
```
执行gcc dumpstack.c -o dumpstack编译并运行，执行结果如下：
```
Stack Track: 
 [0] 0x8048475 
 [1] 0x8048508 
 [2] 0x804855c 
 [3] 0x804856a
```

## 堆溢出
堆, 用来在程序运行时动态分配内存. 当使用`malloc`相关函数, 来分配一块地址空间或初始化一个对象时, 内存地址就是在堆上分配的.

因为堆只是被用来存储数据, 而没有用于存储函数返回地址( returnAddress ), 并且由于堆上的数据的变化,在程序运行中难于监控, 所以攻击者对堆溢出的攻击很难发现.正因如此, 堆溢出备受攻击者青睐.

![Heap overflow](https://developer.apple.com/library/mac/documentation/Security/Conceptual/SecureCodingGuide/Art/heap_overflow.gif)

通常来说, 堆溢出的难度远大于栈溢出. 然而, 现在有很多成功的脚本都涉及堆溢出.堆溢出有两种实现方式:修改数据以及修改对象.

攻击者可以利用堆上的 buffer 溢出来覆盖敏感区域数据.很多语言中, 比如 C++和 Objective-C, 对象(包含函数列表 function vtable和数据指针)是分配在堆上的. 通过利用堆溢出来修改这些指针的方式, 攻击者可以替换成不同的数据, 甚至替换函数的实现. 

利用堆溢出可能极其困难, 但正因如此才更具吸引力, 某些攻击者甚至将此当做了挑战的目标.比如:
1. 解码图片的代码中, 通过堆溢出可以允许远程攻击者执行随意指令
2. 攻击者, 通过构造一个无效的 `"Content-Length" header` 发送 HTTP POST 请求 到存在堆溢出漏洞的网络服务器, 来执行任意指令 




