---
layout: post
title:  "子进程继承了父进程的什么"
subtitle:
date:   2019-6-8T14:25:52-05:00
author: Dandan
header-img: img/post-bg-cook.jpg
catalog: true
categories: C程序技术
tags:
    - C程序技术
---
# 前言
代码从实习算起，目前也是写了一年多了，虽然父子进程之间的关系， 在编程中时不时会遇到，但是有时候整理起来，还是说不上来，今天就做个整理记录一下吧。
# 子进程继承父进程部分
**子进程继承父进程的部分**  
**用户号UIDs和用户组号GIDs**  
**环境Environment**  
**堆栈**  
**共享内存**  
**打开文件的描述符**  
**执行时关闭(Close-on-exec)标志**  
**信号(Signal)控制设定**  
**进程组号**  
**当前工作目录**  
**根目录**  
**文件方式创建屏蔽字**  
**资源限制**  
**控制终端**   
# 子进程独有  
**进程号PID**  
**不同的父进程号**  
**自己的文件描述符和目录流的拷贝**  
**子进程不继承父进程的进程正文（text），数据和其他锁定内存（memory locks）**  
**不继承异步输入和输出**  
  
# 示例以及部分说明
需要注意的是，子进程在继承这些属性和资源时，并不会直接共享父进程中的这些内容，而是复制一份副本。这意味着子进程和父进程在此后可以独立地修改它们各自的副本，而不会相互影响。  

```c
#include <stdio.h>
#include <unistd.h>

int main(){
    int x = 0;
    printf("hello wolrd\n");

    pid_t pid = fork();
    if(pid == -1){
        perror("fork error");
        return -1;
    }else if(pid == 0){
        x = 10;
        printf("child &x = %p, x=%d\n", &x, x);
    }else{
        sleep(1);
        printf("parent &x = %p, x=%d\n", &x, x);
    }

    while(1);
    return 0;
}

```
执行结果
```c
hello wolrd
child &x = 0x7ffc5e25a550, x=10
parent &x = 0x7ffc5e25a550, x=0
```
   从输出结果可以看出，子进程继承了父进程的堆栈，并且可以访问和修改堆栈中的变量。但是子进程和父进程在修改变量时是互不干扰的，它们分别操作自己的堆栈副本。当然包含，父进程中的变量用malloc申请的空间，子进程修改该值，并不影响父进程中该变量的值。所以，使用时，需要在父子进程都释放一遍该空间。

