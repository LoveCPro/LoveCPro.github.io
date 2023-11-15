---
layout: post
title:  vscode和ssh结合使用
subtitle:
date:   2021-7-12T16:25:52-05:00
author: Dandan
catalog: true
header-img: img/post-bg-cook.jpg
tags:
    - Linux
---
# 前言
我开发一直习惯用sourceinsight，后来发现身边人几乎都在用vscode。我一直认为vscode没有sourceinsight方便，例如选择、添加工作区、跳转、布局等等。但是vscode也有很多优点，集成很多插件，功能齐全，主题多样。其中了解到vscode+ssh这个功能，好像很好使，尤其对于虚拟机的开发。之前我都是sourceinsight上编辑代码，然后ssh到远程服务器编译代码，不太方便。vscode+ssh功能就可以集命令行和编辑代码为一体，相对方便一些。

# 安装
## 环境
- ubuntu：18.04
- win7+vscode(1.70)
## ssh
首先ubuntu上需要安上ssh server， 这个就不再赘述了。直接介绍windows 安装ssh客户端。  
- 下载ssh文件
点击打开该链接：https://github.com/PowerShell/Win32-OpenSSH/releases，然后在打开的页面中找到并点击下载OpenSSH-Win64.zip；

- 解压文件
将下载的zip文件解压到 C:\Program Files 路径下；

- 设置环境变量
点击右键“计算机”选择打开：属性->系统->高级系统设置->环境变量。  
然后选择系统变量中的path变量，并在“编辑系统变量”的“变量值”一栏添加 ;C:\Program Files\OpenSSH-Win64 ，注意不同变量值要用分号隔开；

- 测试
打开cmd，然后输入ssh命令并回车，不是现实啥 找不到、not found等，出现的是ssh 参数列表，那就证明安装成功了。

## 配置vscode
- 安装Remote-SSH
打开vscode，然后ctrl+shift+x,然后输入Remote-SSH， 进行安装。

- 配置
    - F1；
    - 然后Remote-SSH,选择add new  SSH Host；
    - 按照格式`ssh user@host`,例如`ssh ddd@192.168.1.1`回车；
    - 显示让选择一个config文件，一般选择在.ssh目录下会自动添加一下信息：
    ```
    Host 192.168.1.1
        HostName 192.168.1.1
        User ddd
    ```
    - 点击vscode左边类电脑图标，就会显示192.168.1.1主机，右击，连接；
    - 会弹出让输入密码即可。

- 连接成功之后后边是编辑区，下边是ssh串口区。可以执行shell命令等。 


> **不能固化习惯，多尝试尝试其他工具**
