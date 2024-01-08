---
layout: post
title:  "select poll epoll区别整理"
date:   2021-12-13T14:25:52-05:00
header-img: img/post-bg-cook.jpg
author: Dandan
catalog: true
tags:
    -  C程序技术
---


# select poll epoll 区别
不管是面试还是平时开发工作都会遇到select poll epoll之间区别的问题，今天大概给总结一下：

| 特性                                 | select                           | poll                             | epoll                             |
|--------------------------------------|----------------------------------|----------------------------------|
| 底层数据结构                         | 数组存储文件描述符               | 链表存储文件描述符              | 红黑树存储文件描述符方式 、双链表存储就绪的文件描述符                        |         
| 如何从fd数据中获取就绪的fd   | 遍历fd_set                       | 遍历链表                         | 回调                             |
| 时间复杂度      |获得就绪的文件描述符需要遍历fd数组 O(n)                             | 获得就绪的文件描述符需要遍历fd链表O(n)                             | 当有就绪事件时，系统注册的回调函数就会被调用，将就绪的fd放到链表中 O(1)                             |
| fd数据拷贝                     | 每次调用select， 需要将fd数据从用户空间拷贝到内核空间     | 每次调用poll需要将fd数据从用户空间拷贝到内核空间     | 使用内存映射mmap,不需要从用户空间频繁拷贝fd数据到内和空间，epoll_ctl时拷贝进内核，之后每次epoll_wait不需拷贝         |
| 内存使用                             | 较高                             | 较高                             | 较低                             |
| 最大连接数限制                       | 有限制，一般为1024               | 无限制                           | 无限制 |



## 应用场景
    a. 连接数较少并且都很活跃,用select和poll效率更高
    b. 连接数较多并且都不很活跃,使用epoll效率更高

## select 缺点：
1.性能开销大：
  1. 调用 select 时会陷入内核，这时需要将参数中的 fd_set 从用户空间拷贝到内核空间
  2. 内核需要遍历传递进来的所有 fd_set 的每一位，不管它们是否就绪
2.文件描述符个数太少

## poll
1.和select几乎没有区别，用户态数组方式传递文件描述符，到了 内核转变成链表存储， 无文件描述符限制， 从性能开销上 和select差别不大
2. selct/poll实现方式（2次拷贝 2次遍历）：
- 将所有连接的socket放入到一个文件描述符集合-----FD_SET(socket_fd, &rd_set)；
- select函数将文件描述符集合---**copy**--->内核；
- 内核**遍历**文件描述符集合，检测到有事件发生，将该socket标记为可读、可写；
- 内核的文件描述符集合---**copy**--->用户态；
- 用户态**遍历**可读可写的socket，然后进行处理---FD_ISSET().

## epoll
1.避免了 开销大的问题  
1.使用红黑树存储文件描述符集合        
2使用队列存储就绪的文件描述符  

适用场景：  
1.连接数很多 且  是 不活跃的连接时，epoll 效率比其他的要高  
2.当连接数较少  且活跃， 使用 select 或者 poll  

# 例子
## select
下边为写的一个 监听内核接口变化的一个例子中摘抄的部分代码：
```c
#include <sys/ipc.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <linux/netlink.h>
#include <linux/rtnetlink.h>


static volatile int keepRunning = 1;

static void intHandler(int dummy)
{
    /*do something*/
}

int main(int argc, char *argv[])
{
    int socket_fd;
    int err = 0;
    fd_set rd_set;
    struct timeval timeout;
    int select_r;
    int read_r;
    struct sockaddr_nl sa;
    struct nlmsghdr *nh;
    int len = BUFLEN;
    char buff[2048] = {0};


    //捕获 CTRL+c信号，使程序安全退出。
    signal(SIGINT, intHandler);

    /*创建套接字，
     * AF_NETLINK表示netlink协议，
     * SOCK_RAW 表示使用原始套接字类型， 能够与底层的数据包交互。
     */
    socket_fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE);
    t_assert(socket_fd > 0);
    //设置套接字， SO_RCVBUF：接收缓冲区大小
    t_assert(!setsockopt(socket_fd, SOL_SOCKET, SO_RCVBUF, &len, sizeof(len)));

    bzero(&sa, sizeof(sa));
    sa.nl_family = AF_NETLINK;
    sa.nl_groups = RTMGRP_LINK | RTMGRP_IPV4_IFADDR | RTMGRP_IPV4_ROUTE;
    //套接字绑定即sa，这个地址配置了要监听的网络事件组，包括网络接口状态变化、IPv4地址变化和路由表变化。
    t_assert(!bind(socket_fd, (struct sockaddr *) &sa, sizeof(sa)));

    while (keepRunning) {
        //初始化文件描述符集合rd_set
        FD_ZERO(&rd_set);
        //将socket_fd添加到集合中，表示要监听socket_fd上的可读事件
        FD_SET(socket_fd, &rd_set);
        timeout.tv_sec = 5;
        timeout.tv_usec = 0;
        //等待套接字上的事件发生即socket_fd， select函数会一直阻塞程序。直到：1.套接字上有程序可读，2.超时。
        select_r = select(socket_fd + 1, &rd_set, NULL, NULL, &timeout);
        /*1.返回值：
         * 正数：正数值表示就绪文件描述符的数量
         * 0：表示在超时时间内没有任何准备就绪描述符， 通常用于超时处理，指示没有时间发生
         * 负数： 表示发生了错误，以便进一步处理。
         */
        if (select_r < 0) {
            perror("select");
        } else if (select_r > 0) {
            //检测是否准备好读取或者写入操作
            if (FD_ISSET(socket_fd, &rd_set)) {
                //有事件发生，read从套接字中读取事件信息，这些信息由内核生成，用于通知用户空间程序网络状态的变化
                read_r = read(socket_fd, buff, sizeof(buff));
                for (nh = (struct nlmsghdr *) buff; NLMSG_OK(nh, read_r); nh = NLMSG_NEXT(nh, read_r)) {
                    /*do something*/

                }
            }
        }
    }

    close(socket_fd);
error:
    if (err < 0) {
        ERR_SYSLOG("Error at line %d\nErrno=%d\n", -err, errno);
    }
    return err;
}
```

## epoll
将上面select代码通过epoll实现一下：
```c
/**
int s = socket(AF_INET, SOCK_STREAM, 0);
bind(s, ...);
listen(s, ...)

int epfd = epoll_create(...);
epoll_ctl(epfd, ...); //将所有需要监听的socket添加到epfd中

while(1) {
    int n = epoll_wait(...);
    for(接收到数据的socket){
        //处理
    }
}
**/
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <sys/epoll.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <linux/netlink.h>
#include <linux/rtnetlink.h>

static volatile int keepRunning = 1;

static void intHandler(int dummy) {
    keepRunning = 0;
}

int main(int argc, char *argv[]) {
    int socket_fd;
    int err = 0;
    struct epoll_event event;
    struct epoll_event events[10];
    int epoll_fd;
    struct sockaddr_nl sa;
    struct nlmsghdr *nh;
    int len = 2048;
    char buff[2048] = {0};

    // 捕获 CTRL+c 信号，使程序安全退出。
    signal(SIGINT, intHandler);

    // 创建套接字
    socket_fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE);
    if (socket_fd < 0) {
        perror("socket");
        return 1;
    }

    // 设置套接字 SO_RCVBUF：接收缓冲区大小
    setsockopt(socket_fd, SOL_SOCKET, SO_RCVBUF, &len, sizeof(len));

    bzero(&sa, sizeof(sa));
    sa.nl_family = AF_NETLINK;
    sa.nl_groups = RTMGRP_LINK | RTMGRP_IPV4_IFADDR | RTMGRP_IPV4_ROUTE;

    // 套接字绑定
    if (bind(socket_fd, (struct sockaddr *) &sa, sizeof(sa)) < 0) {
        perror("bind");
        close(socket_fd);
        return 1;
    }

    // 创建 epoll 实例
    epoll_fd = epoll_create1(0);
    if (epoll_fd == -1) {
        perror("epoll_create1");
        close(socket_fd);
        return 1;
    }

    event.data.fd = socket_fd;
    event.events = EPOLLIN;

    // 添加套接字描述符到 epoll 实例中
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, socket_fd, &event) == -1) {
        perror("epoll_ctl");
        close(socket_fd);
        close(epoll_fd);
        return 1;
    }

    while (keepRunning) {
        int numEvents = epoll_wait(epoll_fd, events, 10, 5000);
        if (numEvents < 0) {
            perror("epoll_wait");
            break;
        } else if (numEvents > 0) {
            for (int i = 0; i < numEvents; i++) {
                if (events[i].data.fd == socket_fd) {
                    int read_r = read(socket_fd, buff, sizeof(buff));
                    if (read_r < 0) {
                        perror("read");
                    } else {
                        for (nh = (struct nlmsghdr *) buff; NLMSG_OK(nh, read_r); nh = NLMSG_NEXT(nh, read_r)) {
                            /* do something */
                        }
                    }
                }
            }
        }
    }

    close(epoll_fd);
    close(socket_fd);
    return 0;
}

```
epoll 的方式即使监听的 Socket 数量越多的时候，效率不会大幅度降低，能够同时监听的 Socket 的数目也非常的多了，上限就为系统定义的进程打开的最大文件描述符个数。因而，epoll 被称为解决 C10K 问题的利器。
下边是从[小林](https://www.xiaolincoding.com/os/8_network_system/selete_poll_epoll.html#epoll)那儿找的图片：
![](/img/epoll_tree.png)
>第一点，epoll 在内核里使用红黑树来跟踪进程所有待检测的文件描述字，把需要监控的 socket 通过 epoll_ctl() 函数加入内核中的红黑树里，红黑树是个高效的数据结构，增删改一般时间复杂度是 O(logn)。而 select/poll 内核里没有类似 epoll 红黑树这种保存所有待检测的 socket 的数据结构，所以 select/poll 每次操作时都传入整个 socket 集合给内核，而 epoll 因为在内核维护了红黑树，可以保存所有待检测的 socket ，所以只需要传入一个待检测的 socket，减少了内核和用户空间大量的数据拷贝和内存分配。

>第二点， epoll 使用事件驱动的机制，内核里**维护了一个链表来记录就绪事件**，当某个 socket 有事件发生时，通过**回调函数**内核会将其加入到这个就绪事件列表中，当用户调用 epoll_wait() 函数时，只会返回有事件发生的文件描述符的个数，不需要像 select/poll 那样轮询扫描整个 socket 集合，大大提高了检测的效率。