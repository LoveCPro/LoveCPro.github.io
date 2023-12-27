---
layout:     post
title:      vpp+wg+ospf(frr)初试
#subtitle:    "\"Hello World, Hello Blog\""
date:       2023-10-28T16:25:52-05:00
author:     Dandan
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - VPP
---
# 前言
如果要在vpp上启用ospf，当前选择的是frrouting。业务模型是![vpp与wg三层业务拓扑](/img/vpp与wg三层业务.png)  
和之前一篇[vpp+wg实现三层转发](https://lovecpro.github.io/2023/10/27/vpp+wg%E5%AE%9E%E7%8E%B0%E4%B8%89%E5%B1%82%E8%BD%AC%E5%8F%91/)一样，不过之前是通过静态路由实现，这次通过动态路由实现。

# 配置
## vpp设备配置

- vpp中接口配置

```
## 打开镜像开关
sudo vppctl lcp lcp-sync on

## 配置wan口，对应的内核接口eth1
sudo vppctl lcp create GigabitEthernet3/0/0 host-if eth1
sudo ip link set dev eth1 up
sudo ip link set mtu 1500 dev eth1
sudo ip address add  10.10.10.1/24   dev eth1

## 配置wg隧道
sudo vppctl wireguard create listen-port 9999 private-key wNw3zMmL/MSvnlIZ+dBnJkHCD5gMEP1HS0cU5gHdhnM= src 10.10.10.1
sudo vppctl lcp create wg0 host-if wg0 tun
sudo vppctl wireguard peer add  wg0 public-key rFHqtOHXmAlhat+xHk3XI1WpFy8CJv87S1XIPjDD1HA=   allowed-ip 0.0.0.0/0   persistent-keepalive 25

sudo ip link set dev wg0 up
sudo ip link set mtu 1420 dev wg0
sudo ip address add 40.40.42.1/30 dev wg0
sudo ip route add 40.40.42.2/32 dev wg0

## 配置lan口， 对应的内核接口是eth2
sudo vppctl lcp create GigabitEthernet1b/0/0 host-if eth2
sudo ip link set dev eth2 up
sudo ip link set mtu 1500 dev eth2
sudo ip address add  172.16.1.3/24 dev eth2

```

- ospf配置
  
```
interface wg0
 ip ospf network non-broadcast
exit
!
router ospf
 ospf router-id 192.168.152.133
 network 40.40.42.0/30 area 0
 network 172.16.1.0/24 area 0
 neighbor 40.40.42.2
exit

```
此处，将wg接口类型改成了NBMA，它默认是点对点的，但是vpp中的wg接口一直发不出去组播报文（除非配一条组播的明细路由），所以采用NBMA。

## CPE配置

- 接口配置  

```shell
  config interface 'wan'
        option type 'ovs-bridge'
        option proto 'static'
        option ipaddr '10.10.10.2'
        option netmask '255.255.255.0'
        option gateway '10.10.10.1'
        list ifname 'eth1'


config interface 'seth2'
        option type 'ovs-bridge'
        option proto 'static'
        option ipaddr '192.168.73.4'
        option netmask '255.255.255.0'
        list ifname 'eth2'

config interface 'wg1'
        option proto 'wireguard'
        option private_key 'CCX+tFOKNPMQg2nhH/7PNGcCp6ycKC/JtX2Y2m4Rw1c='
        list addresses '40.40.42.2/30'

config wireguard_wg1 'wgserver1'
        option public_key 'bzbI5vzSogyEOqlQBeElu7A3kipdlI6NFGdMUzTnzWw='
        option endpoint_host '10.10.10.1'
        option endpoint_port '9999'
        option route_allowed_ips '1'
        option persistent_keepalive '25'
        list allowed_ips '0.0.0.0/0'

  ```

- ospf 配置

```shell
interface wg1
 ip ospf network non-broadcast
!
router ospf
 ospf router-id 192.168.152.136
 network 40.40.42.0/30 area 0
 network 192.168.73.0/24 area 0
 neighbor 40.40.42.1

```
同样，wg接口修改成NBMA。

## 状态

- 邻居正常建立  

```shell  
 show ip ospf  neighbor

Neighbor ID     Pri State           Dead Time Address         Interface                        RXmtL RqstL DBsmL
192.168.152.133   1 Full/Backup       39.867s 40.40.42.1      wg1:40.40.42.2                       0     0     0
```

- 能学习到对端lan侧路由
CPE:学到172.16.1.0/24路由：  

```shell  
show ip ospf route
============ OSPF network routing table ============
N    40.40.42.0/30         [10] area: 0.0.0.0
                           directly attached to wg1
N    172.16.1.0/24         [20] area: 0.0.0.0
                           via 40.40.42.1, wg1
N    192.168.73.0/24       [10] area: 0.0.0.0
                           directly attached to br-seth2

```
HUB:学到192.168.73.0/24路由：
```shell
vpp# show ip ospf  route
============ OSPF network routing table ============
N    40.40.42.0/30         [10] area: 0.0.0.0
                           directly attached to wg0
N    172.16.1.0/24         [10] area: 0.0.0.0
                           directly attached to eth2
N    192.168.73.0/24       [20] area: 0.0.0.0
                           via 40.40.42.2, wg0

```
其间遇到一个问题，HUB上学到的路由是inactive，将frr源码(最新代码)编译正常学习到路由，见文章[源码编译frr](https://lovecpro.github.io/2023/12/11/%E6%BA%90%E7%A0%81%E7%BC%96%E8%AF%91%E5%AE%89%E8%A3%85frrouting/)
