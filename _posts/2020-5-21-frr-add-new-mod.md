---
layout:     post
title:      frr新增模块流程
#subtitle:    "\"Hello World, Hello Blog\""
date:       2020-05-21
author:     Dandan
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - FRR
---
# 前言
来到新的公司后，一直在关注frr的东西，领导让搞一下在frr中新增一个模块，趁着这个机会整理一下整个流程。

# 下载openwrt+frr
- 文档链接
  <http://docs.frrouting.org/projects/dev-guide/en/latest/building-frr-for-openwrt.html>
- 下载代码并设置frr编译环境
```bash
git clone https://github.com/openwrt/openwrt.git
cd openwrt
./scripts/feeds update -a
./scripts/feeds install -a
cd feeds/routing
git fetch origin pull/319/head
git read-tree --prefix=frr/ -u FETCH_HEAD:frr
cd ../../package/feeds/routing/
ln -sv ../../../feeds/routing/frr .
cd ../../..
```  

# 新增模块涉及文件修改
模块名以devset为例。
## 修改模块文件
- 模块编译需要subdir.am、Makefile，内容例如：
![](/img/frr/devset_subdir_am.jpg)![](/img/frr/devset_mkfile.jpg)
- 项目编译需要Makefile.am和configure.ac，内容添加：  
  **configure.ac：**  
  ![](/img/frr/devset_configure_ac1.jpg)![](/img/frr/devset_configure_ac2.jpg)![](/img/frr/devset_configure_ac3.jpg)
**Makefile.am：**
![](/img/frr/devset_conpile_mkfile_am1.jpg)![](/img/frr/devset_conpile_mkfile_am2.jpg)

## 进程文件
改到的文件有frr.in、frrcommon.sh.in, 。
其中进程的方式有：  
1.启动进程zebra时自动开启进程，例如staticd模块；  
2.修改文件damons，通过设置yes或no方式开启进程，例如ripd模块；  
3.例如watchfrr一样，自动开启进程。
### 方式1
- 文件frr.in文件
**添加**
![](/img/frr/dev_frr_in.jpg)
**设置自动启动：**
![](/img/frr/devset_auto_start.jpg)
**关闭进程:**
![](/img/frr/devset_frrin_stop.jpg)
- frrcommon.sh.in文件
 ![](/img/frr/devset_frrcommon_in.jpg) ![](/img/frr/devset_frrcommon_in2.jpg)  
- Frr-reload.py文件
  ![](/img/frr/devset_frr_reload_py.jpg) 
- Daemons文件  
  添加新加模块：  
    ![](/img/frr/devset_daemon.jpg) 

### 方式2
- Frr.in文件
  ![](/img/frr/dev_frr_in.jpg)
- Frrcommon.sh.in文件
 ![](/img/frr/devset_frrcommon_in.jpg)
- Daemons文件
![](/img/frr/devset2_daemon.jpg)

### 方式3  
待研究。

# 程序流程
- frr预初始化：
```c
void frr_preinit(struct frr_daemon_info *daemon, int argc, char **argv)；
```
- 传参处理  
  ![](/img/frr/devset_src_arg.jpg)
  `struct thread_master *frr_init(void)`
- 模块其他功能的初始化操作。（需要则有）  
- Vty初始化
- void frr_config_fork(void)  
-  运行void frr_run(struct thread_master *master)；  
还需要在vtysh上添加模块。  
**文件vtysh.c**  
    ![](/img/frr/devset_vtysh_c.jpg)
**文件vtysh.h**  
 ![](/img/frr/devset_vtysh_h.jpg) 

# openwrt编译文件修改
-  新增模块添加进menuconfig
   在目录./tmp下的文件.config-package.in添加新模块：    
   ![](/img/frr/devset_config_pkage.jpg) 
   执行make menuconfig将新模块添加进去。  

-  frr的makefile添加编译项
Openwrt编译frr相关的文件有feeds/packages/net/frr/Makefile或feeds/routing/frr/Makefile。根据frr版本选择不同目录：
   ![](/img/frr/devset_openwrt_frr_mk.jpg)   
以feeds/routing/frr/Makefile为例  
   ![](/img/frr/devset_feeds_mk1.jpg) 
   ![](/img/frr/devset_feeds_mk2.jpg)   
   ![](/img/frr/devset_feeds_mk3.jpg)    
   ![](/img/frr/devset_feeds_mk4.jpg)   
   ![](/img/frr/devset_feeds_mk5.jpg)   

# 编译配置
执行`make menuconfig`:
   ![](/img/frr/devset_menuconfig1.jpg)  
   ![](/img/frr/devset_menuconfig2.jpg)  

