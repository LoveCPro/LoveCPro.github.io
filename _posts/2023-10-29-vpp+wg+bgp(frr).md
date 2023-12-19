---
layout:     post
title:      vpp+wg+bgp(frr)初试
#subtitle:    "\"Hello World, Hello Blog\""
date:       2023-10-29T16:25:52-05:00
author:     Dandan
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - VPP
---
# 前言
昨天已经就vpp+ospf的结合使用作了总结，同样的，又试了试bgp，是OK的，当然目前都是简单的试试它的功能可用否，并没有做复杂的拓扑上的功能验证。拓扑还是沿用之前拓扑：  
CPE[ lan(192.168.73.4)-----wg(40.40.42.2)-----wan(10.10.10.2) ]-------------------------HUB[ wan(10.0.0.1)-----wg(40.40.42.1)-----lan(172.16.1.3) ] 

# 配置
## vpp设备配置

- vpp中接口配置

```shell
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

- bgp配置

```shell
router bgp 65535
 bgp router-id 192.168.152.133
 neighbor 40.40.42.2 remote-as 65535
 !
 address-family ipv4 unicast
  network 172.16.1.0/24
 exit-address-family
exit

```

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

- bgp 配置

```shell
router bgp 65535
 bgp router-id 192.168.152.136
 neighbor 40.40.42.1 remote-as 65535
 !
 address-family ipv4 unicast
  network 192.168.73.0/24
 exit-address-family

```

## 状态

- 邻居正常建立  

```shell  
# show ip bgp  summary

IPv4 Unicast Summary:
BGP router identifier 192.168.152.136, local AS number 65535 vrf-id 0
BGP table version 2
RIB entries 3, using 552 bytes of memory
Peers 1, using 13 KiB of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
40.40.42.1      4      65535      21      21        0    0    0 00:17:48            1

Total number of neighbors 1

```

- 能学习到对端lan侧路由  
CPE:学到172.16.1.0/24路由：

```shell
# show ip route bgp
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued route, r - rejected route

B>* 172.16.1.0/24 [200/0] via 40.40.42.1, wg1, 00:19:19
```

- ping对端lan侧地址能互通  

```shell
 ping 172.16.1.3 -I 192.168.73.4
PING 172.16.1.3 (172.16.1.3) from 192.168.73.4: 56 data bytes
64 bytes from 172.16.1.3: seq=0 ttl=64 time=0.403 ms
64 bytes from 172.16.1.3: seq=1 ttl=64 time=0.384 ms
64 bytes from 172.16.1.3: seq=2 ttl=64 time=0.360 ms
64 bytes from 172.16.1.3: seq=3 ttl=64 time=0.377 ms
^C
--- 172.16.1.3 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.360/0.381/0.403 ms

```
