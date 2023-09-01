---
layout:     post
title:      gettid和pthread_self区别
#subtitle:    "\"Hello World, Hello Blog\""
date:       2019-07-12
author:     Dandan
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - C程序技术
---
# 二者由来

 - pthread_self 是POSIX的实现，返回值是pthread_t, 无符号长整型，是可移植的库函数
 - gettid是系统调用，返回值是pid_t，无符号整形

1.POSIX是IEEE为了软件能 各种UNIX操作系统上（兼容）运行 而定义的一系列统一的API标准的总称 。它定义了具备可移植操作系统的各种标准，其中就包含线程的概念。
2.linux 内核早期并没有线程的概念，只有进程。后来的线程是一种轻量级的线程。

# 区别
linux中task_struct :
```
struct task_struct {
pid_t pid;	//进程的唯一标识
pid_t tgid;//线程组的领头线程的pid成员的值
}
```
getpid()系统调用返回的是当前进程的tgid值而不是pid值
 1. pthread_self

pthread_self是为了区分同一进程种不同的线程, 是由thread的实现来决定的.返回的是同一个进程中各个线程之间的标识号，对于这个进程内是唯一的，**而不同进程中，每个线程返回的pthread_self可能返回的是一样的**。
返回值对应pthread_create创建线程时的ID(POSIX thread ID)。

 2. gettid
 获取的线程id和pid是有关系的，因为在linux中线程其实也是一个进程(clone)，所以它的线程ID也是pid_t类型。在一个进程中，主线程的线程id和进程id是一样的，**该进程中其他的线程id则在linux系统内是唯一的**，因为linux中线程就是进程，而进程号是唯一的。gettid是不可移植的。
 gettid是用来系统内各个线程间的标识符，由于linux采用轻量级进程实现的，它其实返回的应该是pid号（实际的进程ID--》(内核中的线程的ID).

# 为什么有此现象
	线程库实际上由两部分组成:内核的线程支持+用户态的库支持(glibc)。Linux在早期内核不支持线程的时候，glibc就在库中（用户态）以线程（就是用户态线程）的方式支持多线程了。
	POSIX thread只对用户编程的调用接口作了要求，而对内核接口没有要求。linux上的线程实现就是在内核支持的基础上，以POSIX thread的方式对外封装了接口，所以才会有两个ID的问题。

