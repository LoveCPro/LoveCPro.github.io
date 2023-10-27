---
layout: post
title:  vxlan简单记录
#subtitle:    被Linus称作艺术品的组网神器
date:   2023-10-26T14:25:52-05:00
header-img: img/post-bg-cook.jpg
author: Dandan
categories: Linux
catalog: true
tags:
    - VPN开发
---
# 前言
已经做sdwan好几年了，其中我们的各种组网离不开VXLAN的应用。目前做的一个项目又告一段落了，想写点什么，发现以前的博客记录无非都是对于通用的linux、C的一些总结，对于网络通信方面的记录甚少。一方面觉得我只是利用它组网，作为一个程序员，对于他的内核源码一点都不了解，羞于做总结，认为只是谈兵；另一方面，我只是在sdwan组网中应用过，担心知之甚少。但是还是打算记录下来，随着了解的更深入之后，慢慢补充。

# 应用
VXLAN的主要目的是解决传统VLAN的限制，其中VLAN ID的数量有限（通常为4096个），VXLAN使用24位的VXLAN标识符，从而支持数百万个虚拟网络。  
虽然前期来看vxlan是vlan的一种扩展协议，但是目前vxlan构建隧道能力跟vlan迥然不同了。  
首先看下vxlan的报文：
![](/img/vxlan报文.jpg)


## vxlan组件的概念  
- VNI（VXLAN Network Identifier，VXLAN网络标识符）：VXLAN通过VXLAN ID来标识，其长度为24比特。VXLAN 16M个标签数解决了VLAN标签不足的缺点。

- VTEP（VXLAN Tunnel End Point，VXLAN隧道端点）：VXLAN的边缘设备。VXLAN的相关处理都在VTEP上进行，例如识别以太网数据帧所属的VXLAN、基于VXLAN对数据帧进行二层转发、封装/解封装报文等。VTEP可以是一台独立的物理设备，也可以是虚拟机所在服务器的虚拟交换机。

- VXLAN Tunnel：两个VTEP之间点到点的逻辑隧道。VTEP为数据帧封装VXLAN头、UDP头、IP头后，通过VXLAN隧道将封装后的报文转发给远端VTEP，远端VTEP对其进行解封装。  

## 配置：
ip link命令行：
```
# 创建
ip link add vxlan0 type vxlan id 100 local 192.168.1.10 remote 192.168.1.20 dev eth0 dstport 4789
# 查询
ip -d l show dev vxlan0
# up接口
ip link set vxlan0 up
# 删除
ip link del vxlan0
```
- vxlan0是你要创建的VXLAN接口的名称，你可以根据需要选择不同的名称。
- id 100是VXLAN的标识符，用于唯一标识VXLAN网络。
- local 192.168.1.10是本地端点的IP地址。
- remote 192.168.1.20是远程端点的IP地址。
- dev eth0是底层物理网络接口的名称，这是VXLAN封装的基础网络。
- dstport 4789是VXLAN数据包的目标端口，通常使用默认的4789端口。


openwrt中uci配置：
```
config interface 'vxlan1'
        option proto 'vxlan'
        option port '4789'
        option vid '10001'
        option ipaddr '192.168.2.219'
        option mtu '1500'
        option family_type 'ipv4'
        option peeraddr '192.168.2.234'
        option disabled '0'
```

VxLAN 本质上是一种隧道封装技术。它使用 TCP/IP 协议栈的惯用手法——封装/解封装技术，将 L2 的以太网帧（Ethernet frames）封装成 L4 的 UDP 数据报（datagrams），然后在 L3 的网络中传输，效果就像 L2 的以太网帧在一个广播域中传输一样，实际上是跨越了 L3 网络，但却感知不到 L3 网络的存在。