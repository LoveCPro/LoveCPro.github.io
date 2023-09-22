---
layout: post
title:  记录一次shell脚本代替make  menuconfig的过程
#subtitle:
date:   2022-11-05T16:25:52-05:00
author: Dandan
catalog: true
header-img: img/post-bg-cook.jpg
tags:
    - Linux
---

# 前言
之前做的项目都是通过jenkins平台自动编译的，后来又加入了一个项目，那个代码别人已经搞了很长一段时间了，他们编译以及发布版本都是在本地手动编译，我们用他们的代码搞别的项目，考虑到在本地编译版本发布版本不太规范，所以想着也要要jenkins上编译，这就需要一个自动编译的脚本。

# 实现
之前在本地编译 无非就是 先`make menuconfig`,然后make。`make menuconfig`是一个互动的配置过程， 如果想在jenkins上编译的话，就没有这个互动过程，所以自动编译的脚本就要完全代替make menuconfig的过程。

## make menuconfig
既然要代替make menuconfig过程，首先要要了解下他的构成以及都做了什么。
### 涉及的文件
- Linux内核根目录下的scripts文件夹， 不用关注，是一些图形绘制文件

- arch/$ARCH/Kconfig文件、各层目录下的Kconfig文件

- Linux内核根目录下的makefile文件、各层目录下的makefile文件： 定义环境变量的值

- Linux内核根目录下的的.config文件、arm/$ARCH/下的config文件：系统配置的默认值

- Linux内核根目录下的 include/generated/autoconf.h文件，配置项的宏定义。

### 编译
这就好办了，我们只要替代.config 文件 以及 autoconf.h文件就可以了。在本地代码目录下执行 make menuconfig，将根目录下.config和内核目录下的.config、autoconfig.h 放到相应的目录下。然后看一下make menuconfig的执行过程，看都执行了一些什么过程，这些过程能写的都写到自动编译脚本中，其实不写也可以，因为配置文件已经有了。  
将上述过程结果该上传git上传git，其他过程写入脚本，然后再jenkins中构建就可以了。

# 其他
工作中如果有些事情没有头绪时，且跟之前做的流程不一样时，可以理一理现有的东西，可以走一些“旁门左路”， “耍一些小聪明”，迂回一下说不定也可以解决。就算有之前的经验，但是换一个项目时，完全按照之前的做法说不定也不可行，就要当成个小白，搞些别的路子，哈哈。