---
layout:     post
title:      vrrp(1)---openwrt系统上通过frr实现vrrp小试
#subtitle:    "\"Hello World, Hello Blog\""
date:       2023-6-7T16:25:52-05:00
author:     Dandan
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - FRR
---

# 前言
双机热备是一个大的功能，主要是通过vrrp协议，目前可以通过frr配置，也可以通过`keepalive`功能，之后可能要通过vpp开发，在那个上边也要实现双机热备，先把openwrt上的vrrp验证过程做个记录，下一篇记录通过keepalive实现。
![](/img/openwrt_frr_vrrp.jpg)

# 实例
## dev1配置
eth1配置：
```
config interface 'lan11'
        option ifname 'eth1'
        option proto 'static'
        option netmask '255.255.255.0'
        option ip6assign '60'
        option ipaddr '10.10.10.15'

```
命令行配置 虚拟接口
```
ip link add vrrp4-2-1 link eth1 addrgenmode random type macvlan mode bridge
 ip link set dev vrrp4-2-1 address 00:00:5e:00:01:05
 ip addr add 10.10.10.16/24 dev vrrp4-2-1
 ip link set dev vrrp4-2-1 up
 ```
 frr配置
 ```
 !
interface eth1
 vrrp 5
 vrrp 5 priority 210
 vrrp 5 ip 10.10.10.16
!

 ```

## dev2配置

eth1配置：
```
 config interface 'lan11'
        option ifname 'eth1'
        option proto 'static'
        option netmask '255.255.255.0'
        option ip6assign '60'
        option ipaddr '10.10.10.10'

```
命令行配置 虚拟接口
```
ip link add vrrp4-2-1 link eth1 addrgenmode random type macvlan mode bridge
 ip link set dev vrrp4-2-1 address 00:00:5e:00:01:05
 ip addr add 10.10.10.16/24 dev vrrp4-2-1
 ip link set dev vrrp4-2-1 up
 ```
 frr配置
 ```
 !
interface eth1
 vrrp 5
 vrrp 5 priority 200
 vrrp 5 ip 10.10.10.16
!

 ```

 ## 结果
 dev1:
```
#show vrrp

 Virtual Router ID                    5
 Protocol Version                     3
 Autoconfigured                       No
 Shutdown                             No
 Interface                            eth1
 VRRP interface (v4)                  vrrp4-2-1
 VRRP interface (v6)                  None
 Primary IP (v4)                      10.10.10.15
 Primary IP (v6)                      ::
 Virtual MAC (v4)                     00:00:5e:00:01:05
 Virtual MAC (v6)                     00:00:5e:00:02:05
 Status (v4)                          Master
 Status (v6)                          Initialize
 Priority                             210
 Effective Priority (v4)              210
 Effective Priority (v6)              210
 Preempt Mode                         Yes
 Accept Mode                          Yes
 Advertisement Interval               1000 ms
 Master Advertisement Interval (v4)   1000 ms
 Master Advertisement Interval (v6)   0 ms
 Advertisements Tx (v4)               316
 Advertisements Tx (v6)               0
 Advertisements Rx (v4)               3
 Advertisements Rx (v6)               0
 Gratuitous ARP Tx (v4)               1
 Neigh. Adverts Tx (v6)               0
 State transitions (v4)               2
 State transitions (v6)               0
 Skew Time (v4)                       170 ms
 Skew Time (v6)                       0 ms
 Master Down Interval (v4)            3170 ms
 Master Down Interval (v6)            0 ms
 IPv4 Addresses                       1
 ..................................   10.10.10.16
 IPv6 Addresses                       0

```

dev2:
```
# show vrrp

 Virtual Router ID                    5
 Protocol Version                     3
 Autoconfigured                       No
 Shutdown                             No
 Interface                            eth1
 VRRP interface (v4)                  vrrp4-2-1
 VRRP interface (v6)                  None
 Primary IP (v4)
 Primary IP (v6)                      ::
 Virtual MAC (v4)                     00:00:5e:00:01:05
 Virtual MAC (v6)                     00:00:5e:00:02:05
 Status (v4)                          Backup
 Status (v6)                          Initialize
 Priority                             200
 Effective Priority (v4)              200
 Effective Priority (v6)              200
 Preempt Mode                         Yes
 Accept Mode                          Yes
 Advertisement Interval               1000 ms
 Master Advertisement Interval (v4)   1000 ms
 Master Advertisement Interval (v6)   0 ms
 Advertisements Tx (v4)               167
 Advertisements Tx (v6)               0
 Advertisements Rx (v4)               361
 Advertisements Rx (v6)               0
 Gratuitous ARP Tx (v4)               1
 Neigh. Adverts Tx (v6)               0
 State transitions (v4)               3
 State transitions (v6)               0
 Skew Time (v4)                       210 ms
 Skew Time (v6)                       0 ms
 Master Down Interval (v4)            3210 ms
 Master Down Interval (v6)            0 ms
 IPv4 Addresses                       1
 ..................................   10.10.10.16
 IPv6 Addresses                       0

```

dev1抓包：
```
# tcpdump -i eth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
10:02:18.585748 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:19.585896 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:20.586053 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:21.586205 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:22.586348 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:23.586497 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:24.586646 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:25.586858 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:26.587076 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:27.587318 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:28.587534 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:29.587676 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:30.587892 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:31.588105 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:32.588246 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:33.588390 IP 10.10.10.15 > vrrp.mcast.net: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12

```
dev2抓包：
```
# tcpdump -i eth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
10:02:40.590345 IP 10.10.10.15 > 224.0.0.18: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:41.590439 IP 10.10.10.15 > 224.0.0.18: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:42.591002 IP 10.10.10.15 > 224.0.0.18: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:43.591190 IP 10.10.10.15 > 224.0.0.18: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:44.591401 IP 10.10.10.15 > 224.0.0.18: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:45.591593 IP 10.10.10.15 > 224.0.0.18: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:46.591803 IP 10.10.10.15 > 224.0.0.18: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:47.591981 IP 10.10.10.15 > 224.0.0.18: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:48.592190 IP 10.10.10.15 > 224.0.0.18: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:49.592317 IP 10.10.10.15 > 224.0.0.18: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:50.592505 IP 10.10.10.15 > 224.0.0.18: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:51.592717 IP 10.10.10.15 > 224.0.0.18: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:52.593147 IP 10.10.10.15 > 224.0.0.18: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12
10:02:53.593268 IP 10.10.10.15 > 224.0.0.18: VRRPv3, Advertisement, vrid 5, prio 210, intvl 100cs, length 12

```

