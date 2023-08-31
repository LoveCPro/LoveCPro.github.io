---
layout: post
title:  "ubuntu中samba搭建"
subtitle:
date:   2019-3-15T16:25:52-05:00
author: Dandan
catalog: true
header-img: img/post-bg-cook.jpg
tags:
    - Linux
---
# 前言
作为一个 linux环境开发者， 大家肯定遇到过这种情况， 代码在linux下，如果在虚拟机下使用 vscode或者SourceInsight编辑代码， 会非常的不方便，  所以想在windows系统下看、编辑代码， 那么就需要samba这种工具了。

> Samba是SMB协议的一种实现方法，主要用来实现Linux系统的文件和打印服务。Linux用户通过配置Samba服务器可以实现与windows用户的资源共享。进程smbd和nmbd是Samba的核心，在全部时间运行。


当然， linux系统中使用 vim编辑器也可以正常使用， 通过合理的配置，再编辑器中可以看到代码结构、代码块的跳转、高亮等等的功能， 但是中国人大多数的习惯还是 在windows下看代码， 而且windows下一些编辑器的强大功能是vim无法达到的。

# samba安装

 - 本人使用运行环境：ubuntu 18.04
 - 安装samba服务器
```bash
sudo apt-get install samba samba-common
```
- 将需要分享的目录（例如/home/ubuntu）赋予权限

```bash
sudo chmod 777 /home/ubuntu/ -R
```
- 添加samba用户设置密码

```bash
sudo smbpasswd -a ubuntu
```
 ![Sudo smbpasswd -a ubutun](/assets/doc/samba_pwd.png)
 - 修改samba配置文件
 在 /etc/samba/smb.conf 最后添加如下内容
```bash
[share]
   comment = share work folder
   browseable = yes
   path = /home/ubuntu
  # printable = yes
  # guest ok = no
  # read only = yes
   create mask = 0777
   public = yes
  available = yes
  writable = yes
```
- 重启samba服务器

```bash
sudo service smbd restart 
```

**备注：上述只是本人设置的示例， 具体的目录可以根据自己需要设置**

# 使用
计算机右键===》映射网络驱动器， 文件夹中填入虚拟机ip以及目录  

![在这里插入图片描述](/assets/doc/samba_yingshe.png)  
如上图所示，虚拟机ip：192.168.41.57， share为在/etc/samba/smb.conf中配置的**[share]**， 创建完之后， 会在网络位置侧看道多了个空间：  
![在这里插入图片描述](/assets/doc/samba_wangluo.png)  
这个磁盘可以像D E分区一样正常使用。
如果使用sourceinsight查看代码时，代码位置直接选用该分区即可。