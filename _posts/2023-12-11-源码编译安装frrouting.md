---
layout:     post
title:      源码编译安装frr
#subtitle:    "\"Hello World, Hello Blog\""
date:       2023-12-11T16:25:52-05:00
author:     Dandan
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - FRR
---

# 前言
近期一直在搞vpp，想vpp结合frr使用，但是发现了一个问题，通过隧道之间建立邻居学到的路由一直是`inactive`的状态，想换一个版本试试， 看是不是 版本的问题，但是通过`apt policy frr`发现无可用的版本，在 debian、ubuntu20、ubuntu22上都试了，都没有想要的版本号，只能源码安装了。

# 编译及安装
## 安装依赖
需要安装一些依赖：
```
sudo apt-get install autoconf automake libtool make libreadline-dev texinfo  pkg-config libpam0g-dev libjson-c-dev   bison flex  libc-ares-dev python3-dev python3-sphinx  install-info build-essential libsnmp-dev perl  libcap-dev python2 libelf-dev libunwind-dev
```
frr生成Makefile过程中还需要：protobuf
```
sudo apt-get install protobuf-c-compiler  libprotobuf-c-dev
```

编译libyang需要cmake工具以及libpcre2-dev工具包， 命令如下：
```
sudo apt-get install cmake libpcre2-dev
```

## 编译安装libyang
```
git clone https://github.com/CESNET/libyang.git
 
cd libyang
 
git checkout v2.0.0
 
mkdir build
 
cd build
 
cmake -D CMAKE_INSTALL_PREFIX:PATH=/usr \
      -D CMAKE_BUILD_TYPE:String="Release" ..
 
make
 
sudo make install
```

## 编译安装frr

```
git clone https://github.com/frrouting/frr.git frr
 
cd frr
 
./bootstrap.sh
 
./configure \
    --bindir=/usr/bin \
    --sbindir=/usr/lib/frr \
    --sysconfdir=/etc/frr \
    --libdir=/usr/lib/frr \
    --libexecdir=/usr/lib/frr \
    --localstatedir=/var/run/frr \
    --with-moduledir=/usr/lib/frr/modules \
    --enable-user=frr \
    --enable-group=frr \
    --enable-vty-group=frrvty \
    --enable-systemd=yes \
    --disable-exampledir \
    --disable-ldpd \
    --enable-fpm \
    --with-libyang-pluginsdir=/usr/lib/x86_64-linux-gnu\
    --with-pkg-git-version \
    --with-pkg-extra-version=-MyOwnFRRVersion \
    SPHINXBUILD=/usr/bin/sphinx-build

make
 
sudo make install
```

1.其中configure命令会有问题，报错如下：  
```
configure: error: libyang needs to be compiled with ENABLE_LYD_PRIV=ON. Instructions for this are included in the build documentation for your platform at http://docs.frrouting.org/projects/dev-guide/en/latest/building.html
```  

- 解决办法1：  
libyang cmake时，添加`ENABLE_LYD_PRIV=ON`, 如：    
`cmake -D CMAKE_INSTALL_PREFIX:PATH=/usr -D ENABLE_LYD_PRIV=ON  -D CMAKE_BUILD_TYPE:String="Release" ..`  
 
- 解决办法2：  
 CMakeLists.txt 添加`set(ENABLE_LYD_PRIV ON)`, 然后重新将libyang cmake、make、make install。  

2.起frr服务时报错：`Failed to restart frr.service: Unit frr.service is masked`  
该报错表示重新启动 frr.service 服务时，该服务被标记为 "masked"。服务被标记为 "masked" 通常意味着系统不允许启动或停止该服务。解决如下：  
- 解除服务的标记：`sudo systemctl unmask frr.service`
- 重新启动: `sudo systemctl restart frr.service`