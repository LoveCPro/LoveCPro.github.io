---
layout: post
title:  "openwrt编译步骤"
date:   2020-6-2T16:25:52-05:00
header-img: img/post-bg-cook.jpg
author: Dandan
catalog: true
categories: openwrt
tags:
    -  openwrt
---
# 前言
年初入职了新的公司， 是搞通信的， 开发需要基于openwrt系统开发， 初期编译以及使用openwrt系统遇到了各种坑，整理出来，以防入坑。

# 编译
- ubuntu：18.04
- openwrt：19.07

## 安装依赖项
```bash
sudo apt-get install -y bison flex unzip gcc g++ libncurses5-dev zlib1g-dev bison flex  -y --fix-missing
sudo apt-get install autoconf gawk make gettext binutils patch bzip2 libz-dev subversion asciidoc texinfo -y --fix-missing
sudo apt-get install sharutils ncurses-term git-core npm automake -y --fix-missing
```
## 编译操作
使用git下载，openwrt的源码， 进入目录，然后切换到 19.07上, 当然我不是用的以下链接， 用的我们项目自己的仓库。
``bash
git clone https://github.com/openwrt/openwrt.git
```
然后执行两条命令：
```bash
./scripts/feeds update -a
./scripts/feeds install -a
```
这两条命令是根据feeds.conf.default文件内容去下载所需软件包。  
然后执行make menuconfig选择自己的编译项。选择完退出保存， 执行```make V=s```开始编译。

## 快捷方法以及问题处理
1.编译的时候一些软件包需要从网上下载，会非常慢，可以从别人那儿拷贝安装包然后放到dl目录下即可。  
2.如果编译项跟其他人一样， 只需要拷贝其他人的.config文件放到编译目录中， 就可以代码make menuconfig选择的过程。  
3.我用的是ubuntu18.04，项目中有一个软件包依赖nodejs ， 但是 ubuntu18.04默认安装的版本太低，解决办法如下：
网上下载解压node-v10.9.0-linux-x64 安装包 ， 然后删除/usr/bin目录下的node 和 npm，将node-v10.9.0-linux-x64/bin下的node、npm软连接到/usr/bin目录下
4.有一个功能用到了go环境，下载go，并设置go代理， 将go环境设置如下：
```
$ go env
GO111MODULE=''
GOARCH='amd64'
GOBIN=''
GOCACHE='~/.cache/go-build'
GOENV='~/.config/go/env'
GOEXE=''
GOEXPERIMENT=''
GOFLAGS=''
GOHOSTARCH='amd64'
GOHOSTOS='linux'
GOINSECURE=''
GOMODCACHE='~/work/go/pkg/mod'
GONOPROXY=''
GONOSUMDB=''
GOOS='linux'
GOPATH='~/work/go'
GOPRIVATE=''
GOPROXY='https://mirrors.aliyun.com/goproxy/'
GOROOT='/snap/go/10319'
GOSUMDB='sum.golang.org'
GOTMPDIR=''
GOTOOLCHAIN='auto'
GOTOOLDIR='/snap/go/10319/pkg/tool/linux_amd64'
GOVCS=''
GOVERSION='go1.21.0'
GCCGO='gccgo'
GOAMD64='v1'
AR='ar'
CC='gcc'
CXX='g++'
CGO_ENABLED='1'
GOMOD='/dev/null'
GOWORK=''
CGO_CFLAGS='-O2 -g'
CGO_CPPFLAGS=''
CGO_CXXFLAGS='-O2 -g'
CGO_FFLAGS='-O2 -g'
CGO_LDFLAGS='-O2 -g'
PKG_CONFIG='pkg-config'
GOGCCFLAGS='-fPIC -m64 -pthread -Wl,--no-gc-sections -fmessage-length=0 -ffile-prefix-map=/tmp/go-build1523107131=/tmp/go-build -gno-record-gcc-switches'
```
