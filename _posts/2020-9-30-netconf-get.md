---
layout: post
title:  "NETCONF浅解"
date:   2020-9-30T16:25:52-05:00
header-img: img/post-bg-cook.jpg
author: Dandan
categories: openwrt
tags:
    -  openwrt
---

## netconf 浅解
由于作者使用netconf中都是用的callhome方式，所以本文默认都是 callhome连接方式，以下不再赘述。
### netconf server  tcp  xml

```c
<netconf-server xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-server">
  <listen>
    <endpoint>
      <name>default-ssh</name>
      <ssh>
        <tcp-server-parameters>
          <local-address>0.0.0.0</local-address>
          <keepalives>
            <idle-time>1</idle-time>
            <max-probes>10</max-probes>
            <probe-interval>5</probe-interval>
          </keepalives>
        </tcp-server-parameters>

```
### 相应代码
xml中的`<idle-time> `对应的以下的 TCP_KEEPIDLE， `<max-probes>`对应TCP_KEEPCNT，`<probe-interval>` 对应的TCP_KEEPINTVL
```c
int
nc_sock_enable_keepalive(int sock, struct nc_keepalives *ka)
{
    int opt;

    opt = 1;
    if (setsockopt(sock, SOL_SOCKET, SO_KEEPALIVE, &opt, sizeof opt) == -1) {
        ERR("Could not set SO_KEEPALIVE option (%s).", strerror(errno));
        return -1;
    }
#if 0
    if (!ka->enabled) {
        return 0;
    }
#endif

#ifdef TCP_KEEPIDLE
    opt = ka->idle_time;
    if (setsockopt(sock, IPPROTO_TCP, TCP_KEEPIDLE, &opt, sizeof opt) == -1) {
        ERR("Setsockopt failed (%s).", strerror(errno));
        return -1;
    }
#endif

#ifdef TCP_KEEPCNT
    opt = ka->max_probes;
    if (setsockopt(sock, IPPROTO_TCP, TCP_KEEPCNT, &opt, sizeof opt) == -1) {
        ERR("Setsockopt failed (%s).", strerror(errno));
        return -1;
    }
#endif

#ifdef TCP_KEEPINTVL
    opt = ka->probe_interval;
    if (setsockopt(sock, IPPROTO_TCP, TCP_KEEPINTVL, &opt, sizeof opt) == -1) {
        ERR("Setsockopt failed (%s).", strerror(errno));
        return -1;
    }
#endif
#if 1
    opt = 20000;
    if (setsockopt(sock, IPPROTO_TCP, TCP_USER_TIMEOUT, &opt, sizeof opt) == -1) {
        ERR("Setsockopt failed (%s).", strerror(errno));
    }
#endif

    return 0;
}

```


[RFC1122#TCP KEEP-ALIVE](https://link.zhihu.com/?target=https://tools.ietf.org/html/rfc1122#page-101)

> KeepAlive并不是TCP协议规范的一部分，但在几乎所有的TCP/IP协议栈（不管是Linux还是Windows）中，都实现了KeepAlive功能。
> 先来看看KeepAlive都支持哪些设置项

KeepAlive默认情况下是关闭的，可以被上层应用开启和关闭
**tcp_keepalive_time**: KeepAlive的空闲时长，或者说每次正常发送心跳的周期，默认值为7200s（2小时）
**tcp_keepalive_intvl**: KeepAlive探测包的发送间隔，默认值为75s
**tcp_keepalive_probes**: 在tcp_keepalive_time之后，没有接收到对方确认，继续发送保活探测包次数，默认值为9（次）

在各个地方设置方式如下：
####  linux内核
查看配置：

```clike
cat /proc/sys/net/ipv4/tcp_keepalive_time
cat /proc/sys/net/ipv4/tcp_keepalive_intvl
cat /proc/sys/net/ipv4/tcp_keepalive_probes
```
对应sysctl.conf文件：

```clike
net.ipv4.tcp_keepalive_time=7200
net.ipv4.tcp_keepalive_intvl=75
net.ipv4.tcp_keepalive_probes=9
```

可以通过 sysctl  -w 命令 配置 ， sysctl -a 命令配置。
####  C语言

```c
#include <sys/socket.h>

int setsockopt(int socket, int level, int option_name,
     const void *option_value, socklen_t option_len);
```
setsockopt配置，参数：
第一个参数是要设置的套接字
第二个参数是SOL_SOCKET
第三个参数必须是SO_KEEPALIVE
第四个参数必须是一个布尔整型值，0表示关闭，1表示打开
最后一个参数是第四个参数值的大小。
