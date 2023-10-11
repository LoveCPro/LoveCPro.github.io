---
layout: post
title:  修复wireguard 包统计源码问题的记录
#subtitle:    被Linus称作艺术品的组网神器
date:   2023-5-9T14:25:52-05:00
header-img: img/post-bg-cook.jpg
author: Dandan
categories: Linux
catalog: true
tags:
    -  VPN开发
---
# 问题现象
这段时间有一个同事做了一个功能，统计接口的收发包速率，他是通过统计接口的收发包大小，然后除以时间得到速率。在测试阶段经常发现wg接口接收方向的包drop了非常多，导致统计跟实际不符。
![](/img/wg_rx_drop.jpg)
经过以下排查手段：  
1.确认业务是否正常。虽然接口统计大量丢包， 但是通过打流发现流正常接收，并没有丢  
2.将drop的数据也统计入收包---不可行，drop是包个数，我们要统计的是包大小。  
然后我就开始怀疑源码是不是有问题，因为业务并没有丢包，所有的现象均正常，只是接口显示不正常。  
到wg目录，有一个专门的reviced.c，尝试搜drop能不能找到，果然，一下子就找到了：
```
++dev->stats.rx_dropped;
```  
阅读代码后，发现甚是不合理，果断给去掉。

>**勇于怀疑源码、修改源码** 
