---
layout: post
title:  ubuntu设置静态地址
date:   2020-1-15T14:25:52-05:00
header-img: img/post-bg-cook.jpg
author: Dandan
categories: Linux
catalog: true
tags:
    -  Linux
---
# 前言
之前公司的开发环境有用ubuntu、centos，到了现在的新公司，基本上都是用的ubuntu，新搞一个ubuntu开发环境时想要配置静态ip总要查询资料，有的还不适用，今天就整理一下桥接和NAT模式下都是什么配置的。

# NAT网络的准备工作
环境：ubuntu18.04
```
cat /etc/os-release
NAME="Ubuntu"
VERSION="18.04.3 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.3 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic

```
## 先查看ip是不是动态的
`ip addr`命令查询对应的网卡 ip那行有dynamic字样就说明是动态地址
## 查找网关地址
“编辑” ---》 网络编辑器---》选中对应的网络（外部链接是NAT模式）---》NAT设置， 看到有一个网关ip，记录这个就是一会要设置的网关地址。 

# NAT/桥接配置
如果是桥接的话一样需要执行`ip addr`看一下地址是动态还是静态的。
## 查看配置网卡文件
```
ls /etc/netplan/                                                                         
01-network-manager-all.yaml 
```
我的虚拟机上是01-network-manager-all.yaml ， 在其它服务器、云主机等文件名可能不一样，但是一般`/etc/netplan/`目录下只有一个文件，名称格式是*.yaml，查看文件内容：  
```
cat 01-network-manager-all.yaml                                                         
# Let NetworkManager manage all devices on this system
network:
  version: 2
  renderer: NetworkManager
```

## 修改配置文件
格式如下：
```
cat 01-network-manager-all.yaml                                                        
# Let NetworkManager manage all devices on this system
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:   # 对应第1步查到的网络接口名称
      addresses:
        - 192.168.242.172/24   # 自己想要配置的静态 ip
      gateway4: 192.168.242.2  # 对应上面查到的网关 ip
      nameservers:
          addresses: [192.168.242.2, 8.8.8.8]

```

## 应用配置
生效命令：
```
netplan apply
```
*ip addr*查看配置，如果没有dynamic字样，说明配置成了静态。*ping www.baidu.com*测试修改情况。

