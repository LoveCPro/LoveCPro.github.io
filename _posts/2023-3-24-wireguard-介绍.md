---
layout: post
title:  wireguard介绍
subtitle:    被Linus称作艺术品的组网神器
date:   2023-3-24T14:25:52-05:00
header-img: img/post-bg-cook.jpg
author: Dandan
categories: Linux
catalog: true
tags:
    -  VPN开发
---
# 前言
从2021年开始， sdwan业务打算引入wg。之前我所在公司sdwan业务中vpn技术一直用的是ipsec， 但是设备原因ipsec在我们的业务上线后暴露了很多的问题，所以期望wg能解决这些问题。当时该开发任务交给了我。wg业务已经做了几年了，目前也不是很忙，领导让给大家做一下wg相关培训，刚好趁着这个机会把wg整理一下。

# 简述
## wg优势
Wireguard被视为下一代vpn协议，旨在解决许多困扰IPSec/IKEv2、OpenVPN 或 L2TP 等其他 VPN 协议的问题。旨在比ipsec更快、更精简、更有用，避免令人头疼的问题；旨在比OPenvpn具备更高的性能。
WireGuard 与其他 VPN 协议的性能测试对比：  
![](/img/wg与其他的对比.jpg)  
代码量对比：
![](/img/wg代码量对比.jpg)  
Openvpn有10万行代码，而wireguard只有大概4000行代码，代码库相当精简。可以看到wireguard 直接碾压其他VPN协议。  
综上，Wireguard优点：  
- 配置精简，可直接使用默认值
- 只需最少的密钥管理工作，每个主机只需要 1 个公钥和 1 个私钥。
- 就像普通的以太网接口一样，以 Linux 内核模块的形式运行，资源占用小。
- 能够将部分流量或所有流量通过 VPN 传送到局域网内的任意主机。
- 能够在网络故障恢复之后自动重连，戳到了其他 VPN 的痛处。
- 比目前主流的 VPN 协议，连接速度要更快，延迟更低（见上图）。
- 使用了更先进的加密技术，具有前向加密和抗降级攻击的能力。
- 可以运行在主机中为容器之间提供通信，也可以运行在容器中为主机之间提供通信。
如此优秀的VPN技术，Linus本尊对WG赞不绝口，他在18年一封邮件中称其为一件艺术品：work of art。  

## 我司为何选用wg
在wireguard之前我司都是用的ipsec，遇到了以下的问题：
- Ipsec建立慢；
- 不能与在同一内网的设备建立ipsec；
- 过一段时间ipsec自动更新秘钥；
- Ipsec在我司某一款设备中应用性能较低。  

在使用wireguard之后，我司的相关业务应用得到了大幅度的优化以及提升：
- wg隧道建立快，当wg接口up后，隧道就会立马建立，秒级；
- 未出现位于同一内网下的不同设备不能建立的情况；
- Wg的秘钥是固定的
- 性能大幅度提高。

# wg交互报文
WG的交互过程如下：
![](/img/wg交互过程.jpg)  
## 隧道的建立
Wg因为两端都配了对方的公钥，因此可以使用1个RTT，2个报文就可以完成隧道的建立。  
### 握手请求
握手请求报文携带：
- unencrypted_ephemeral：发送方为这次握手临时生成的公钥（未加密，用于 ECDH）
- encrypted_static：用对端公钥和临时生成的私钥 ECDH 出的临时密钥 key1 对称加密对方的公钥
- encrypted_timestamp：用对端公钥和自己的私钥 ECDH 出 key2，key2 混淆进 key1，来加密当前的时间戳
- mac1：对端公钥加上整个报文内容后的哈希
![](/img/wg握手请求报文.jpg)

### 握手响应
接收方校验过程：mac1 ------ encrypted_static  ------  encrypted_timestamp.  
一切ok，接收方生成自己的临时密钥对。此时，接收方因为有了对端的临时公钥，已经可以计算出此次协商后加密数据要用的密钥。但它还需要发送一个握手的回复报文来把自己的临时公钥给发送方以便于发送方可以算出同样的密钥：
- unencrypted_ephemeral：接收方为这次握手临时生成的公钥（未加密，用于 ECDH）
- mac1：对端公钥加上整个报文内容后的哈希
![](/img/wg握手响应报文.jpg)  
两端都有对方临时生成的公钥，加上自己临时生成的私钥，就可以得到这次握手的两个方向的对称加密的密钥。  
如果发送方没有接收到握手回应报文，会继续发送握手请求报文，时间间隔为 5s+jitter（最长333ms）:
![](/img/wg握手间隔.jpg)  

## 数据传输
**发送**  
- 用户态：应用程序发送目标地址是 VPN 对端网络的数据报文
- 内核：内核通过路由表发现应该由 wg0 接口发出，所以交给 WG 处理
- WG：通过目标地址，在接口的配置中可以反查出要发往哪个 peer，然后用之前和该 peer 协商好的密钥（如果没有协商或者密钥过期，则重新协商）加密报文，并将报文封装在目标地址和目标端口是 peer 的 endpoint 的 UDP 报文中（报文中包含 key_index）      
  
**接收：**
- 内核：数据报文的 UDP 端口是 WG 在监听，将其送给 WG 处理
- WG：从报文的 key_index 找到哈希表中对应的密钥，解密（这里不是直接解密，而是放入一个解密队列中，这是设计上网络系统的一个小诀窍）
- WG：查看解密出来的原始报文是否在 peer 允许的 IP 列表中，如果是，就把原始报文交给内核处理。注意，这里这个报文属于哪个 peer，也是从 key_index 中获得
- 内核：根据原始报文的目标地址查路由表将报文送出

**保活机制：**  
保活在wireguard中是可选可配。如果连接是从一个位于 NAT 后面的对等节点（peer）到一个公网可达的对等节点（peer），那么 NAT 后面的对等节点（peer）必须定期发送一个出站 ping 包来检查连通性，如果 IP 有变化，就会自动更新Endpoint。  
将keepalive时间配置为5，keepalive报文就是每5s发送一次（当隧道中无数据传输时）。
![](/img/wgkpalive间隔.jpg)  

# wg配置
WG定义了一个很重要的概念 —— WireGuard Interface（简称 wgi）:
- 有一个自己的私钥
- 有一个用于监听数据的 UDP 端口
- 有一组 peer（peer 是另一个重要的概念），每个 peer 通过该 peer 的公钥确认身份  

**客户端配置：**
```
[Interface]
PrivateKey=2No7YStLNZN6QF4HHkb+5LZpSZt5Ih7aDF+CVMMhsUc=
[Peer]
PublicKey=vHZv3rf4GZsALwEoOml/JugdrI4GRPE5mv3xsWDYbCQ=
PresharedKey=Yh8rHLu5OqBGOqI0j4k/Z/X3VVUxfeYu9FRJ/wMV9B8=
AllowedIPs=0.0.0.0/0
Endpoint=192.168.21.219:51820
PersistentKeepalive=25
```
**服务端配置：**
```
[Interface]
PrivateKey=AHmnxFxHlhEzTt3AUPVKj/b77ocawptL/t1yl2swbGE=
ListenPort=51820
[Peer]
PublicKey=KNWOiEhjRO2my/Rc9KE64VZL1A9TI/f2KmpHDDSoLW4=
PresharedKey=Yh8rHLu5OqBGOqI0j4k/Z/X3VVUxfeYu9FRJ/wMV9B8=
AllowedIPs=44.40.40.2/32
```
**生效（在客户端查看为例）：**
```
# wg show
interface: wg_10004
  public key: KNWOiEhjRO2my/Rc9KE64VZL1A9TI/f2KmpHDDSoLW4=
  private key: (hidden)
  listening port: 51138

peer: vHZv3rf4GZsALwEoOml/JugdrI4GRPE5mv3xsWDYbCQ=
  preshared key: (hidden)
  endpoint: 192.168.21.219:51820
  allowed ips: 0.0.0.0/0
  latest handshake: 1 minute, 22 seconds ago
  transfer: 184 B received, 616 B sent
  persistent keepalive: every 25 seconds
```
**秘钥对生成命令：**
```bash
# Generate keys
umask go=
wg genkey | tee wgserver.key | wg pubkey > wgserver.pub
wg genpsk > wgserver.psk
```

# 扩展延伸

**延伸阅读：**  
https://courses.csail.mit.edu/6.857/2018/project/He-Xu-Xu-WireGuard.pdf  
https://zhuanlan.zhihu.com/p/447375895  
https://zhuanlan.zhihu.com/p/91383212  
https://github.com/wangyu-/udp2raw  
https://github.com/wangyu-/UDPspeeder  
https://github-wiki-see.page/m/maxwellhouse102/UDPspeeder/wiki/Fine-grained-FEC-Parameters  

**名词解释：**
- RTT(Round-Trip Time)，往返时延。在计算机网络中它是一个重要的性能指标，表示从发送端发送数据开始，到发送端收到来自接收端的确认（接收端收到数据后便立即发送确认），总共经历的时延。
- Jitter:随机抖动