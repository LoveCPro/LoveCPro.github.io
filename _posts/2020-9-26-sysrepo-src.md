---
layout: post
title:  "sysrepo源码解析"
date:   2020-9-26T16:25:52-05:00
author: Dandan
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    -  openwrt
---
# 前言
Sysrepo 是一个用于网络设备的轻量级、模块化的配置和状态数据存储库。它以高效的方式存储、管理和检索设备的配置数据，同时支持实时监控设备的运行状态。

## 主要特性

- **模块化架构**：Sysrepo 的设计理念是模块化的，每个模块负责特定设备功能的配置和状态管理。这使得系统的可维护性和可扩展性得以提高。

- **YANG 数据模型**：Sysrepo 使用 YANG 数据建模语言来描述设备的配置和状态。YANG 提供了一种标准化的方式来定义设备的数据模型，使得不同厂家的设备都可以基于共同的规范进行管理。

- **快速读写操作**：Sysrepo 以高性能处理读写操作，确保了实时配置更改和状态监测的效率。

- **事务支持**：Sysrepo 支持事务，保证了在配置更改过程中的一致性和可回滚性。

- **通用 API**：Sysrepo 提供了易于使用的 API，供开发人员构建自定义的网络管理应用程序。

- **与各种协议兼容**：Sysrepo 支持 RESTCONF、NETCONF 等标准协议，使得网络管理可以基于通用的网络协议进行。

## 应用领域

- **网络设备管理**：Sysrepo 可用于管理路由器、交换机等各种网络设备的配置和状态信息。

- **SDN 控制器**：在软件定义网络中，Sysrepo 可以作为 SDN 控制器的基础，实现集中式的网络管理。

- **网络自动化**：Sysrepo 可以与自动化工具集成，实现网络配置的自动化部署和维护。
## 一、sysrepo 源码分析
应用程序通过将对 sysrepo 的调用通过 sysrepo 提供的相应的 API 接口访问方法，称为 syrepo 的间接访问方法。该方法是应用程序通过创建 Deamon 进程，通过 IPC Shm 机制与 sysrepo 通信。可以做到对 sysrepo 的即插即用，最后由 sysrepo 纳管，这就是 Plugind，命名为 sysrepo-plugind。要快速的使用 sysrepo，并快速开发出适配于 sysrepo 的插件，就要先了解 sysrepo-plugind 的实现原理与机制，就需要先从实现 sysrepo-plugind 的源码处着手。sysrepo 源码路径：git clone https://github.com/sysrepo/sysrepo.git 。 Sysrepo-plugind 实现的路径为 sysrepo/src/executables/sysrepo-plugind.c 。下面也就从该文件开始说。

### 1.1 数据结构

```c
/** protected flag for terminating sysrepo-plugind */
int loop_finish;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;

struct srpd_plugin_s {
    void *handle;
    srp_init_cb_t init_cb;
    srp_cleanup_cb_t cleanup_cb;
    void *private_data;
};

```
### 1.2 main函数入口

```c
int
main(int argc, char** argv)
{
    struct srpd_plugin_s *plugins = NULL;
    sr_conn_ctx_t *conn = NULL;
    sr_session_ctx_t *sess = NULL;
    sr_log_level_t log_level = SR_LL_ERR;
    int plugin_count = 0, i, r, rc = EXIT_FAILURE, opt, debug = 0;
    struct option options[] = {
        {"help",      no_argument,       NULL, 'h'},
        {"version",   no_argument,       NULL, 'V'},
        {"verbosity", required_argument, NULL, 'v'},
        {"debug",     no_argument,       NULL, 'd'},
        {NULL,        0,                 NULL, 0},
    };

    /* 解析sysrepo-plugind的参数，默认是 sysrepo-plugind -d -v 0  */
    opterr = 0;
    while ((opt = getopt_long(argc, argv, "hVv:d", options, NULL)) != -1) {
        switch (opt) {
        case 'h':
            version_print();
            help_print();
            rc = EXIT_SUCCESS;
            goto cleanup;
        case 'V':
            version_print();
            rc = EXIT_SUCCESS;
            goto cleanup;
        case 'v':
            if (!strcmp(optarg, "none")) {
                log_level = SR_LL_NONE;
            } else if (!strcmp(optarg, "error")) {
                log_level = SR_LL_ERR;
            } else if (!strcmp(optarg, "warning")) {
                log_level = SR_LL_WRN;
            } else if (!strcmp(optarg, "info")) {
                log_level = SR_LL_INF;
            } else if (!strcmp(optarg, "debug")) {
                log_level = SR_LL_DBG;
            } else if ((strlen(optarg) == 1) && (optarg[0] >= '0') && (optarg[0] <= '4')) {
                log_level = atoi(optarg);
            } else {
                error_print(0, "Invalid verbosity \"%s\"", optarg);
                goto cleanup;
            }
            break;
        case 'd':
            debug = 1;
            break;
        default:
            error_print(0, "Invalid option or missing argument: -%c", optopt);
            goto cleanup;
        }
    }

    /* check for additional argument */
    if (optind < argc) {
        error_print(0, "Redundant parameters");
        goto cleanup;
    }

    /* 加载插件库 ，默认在 /usr/lib/sysrepo/plugins */
    if (load_plugins(&plugins, &plugin_count)) {
        goto cleanup;
    }

    /* daemonize, sysrepo-plugind no longer directly logs to stderr */
    daemon_init(debug, log_level);

    /* create connection (after we have forked so that our PID is correct) */
    if ((r = sr_connect(0, &conn)) != SR_ERR_OK) {
        error_print(r, "Failed to connect");
        goto cleanup;
    }

    /* create session */
    /*调用sysrepo API(sr_session_start)创建与sysrepo running库的会话，并启动该会话*/
    if ((r = sr_session_start(conn, SR_DS_RUNNING, &sess)) != SR_ERR_OK) {
        error_print(r, "Failed to start new session");
        goto cleanup;
    }

    /* init plugins */
    /*对所有已加载的plugin通过调用init_cb注册的回调初始化，这是整个main第二处核心点，与用户是强 
     *相关用户开发的插件，注册，订阅，初始化都通过init_cb，否则不能将sysrepol通信连接*/
    for (i = 0; i < plugin_count; ++i) {
        r = plugins[i].init_cb(sess, &plugins[i].private_data);
        if (r != SR_ERR_OK) {
            SRP_LOG_ERR("Plugin initialization failed (%s).", sr_strerror(r));
            goto cleanup;
        }
    }

    /* wait for a terminating signal */
    pthread_mutex_lock(&lock);
    while (!loop_finish) {
        pthread_cond_wait(&cond, &lock);
    }
    pthread_mutex_unlock(&lock);

    /* cleanup plugins */
    /*进程结束后，回收初始申请的资源，如果不进行回收，部分版本会报段错误*/
    for (i = 0; i < plugin_count; ++i) {
        plugins[i].cleanup_cb(sess, plugins[i].private_data);
    }

    /* success */
    rc = EXIT_SUCCESS;

cleanup:
    for (i = 0; i < plugin_count; ++i) {
        dlclose(plugins[i].handle);
    }
    free(plugins);

    sr_disconnect(conn);
    return rc;
}

```


### 1.3 load_plugin

```c

static int
load_plugins(struct srpd_plugin_s **plugins, int *plugin_count)
{
    void *mem, *handle;
    DIR *dir;
    struct dirent *ent;
    const char *plugins_dir;
    char *path;
    int rc = 0;

    *plugins = NULL;
    *plugin_count = 0;

    /* get plugins dir from environment variable, or use default one */
    /*获取路径，在@PLUGINS_PATH@在CMakeList.txt中定义，在编译时也可以自定义*/
    plugins_dir = getenv("SRPD_PLUGINS_PATH");
    if (!plugins_dir) {
        plugins_dir = SRPD_PLUGINS_PATH;
    }

    /* create the directory if it does not exist */
    /*access函数，:检查调用进程是否可以对指定的文件执行某种操作， F_OK文件是否存在
     * 本段代码是检测SRPD_PLUGINS_PATH目录是否存在，如果不存在，调用sr_mkpath创建目录，并设置*            
     * 目录的访问权限000777。本段代码是安全性代码，确保指定的路径存在。对于实际开发中，是通过编 
     * 译是指定，不存在路径的动态库无法安装。*/
    if (access(plugins_dir, F_OK) == -1) {
        if (errno != ENOENT) {
            error_print(0, "Checking plugins dir existence failed (%s).", strerror(errno));
            return -1;
        }
        if (sr_mkpath(plugins_dir, 00777) == -1) {
            error_print(0, "Creating plugins dir \"%s\" failed (%s).", plugins_dir, strerror(errno));
            return -1;
        }
    }

    dir = opendir(plugins_dir);
    if (!dir) {
        error_print(0, "Opening \"%s\" directory failed (%s).", plugins_dir, strerror(errno));
        return -1;
    }

    while ((ent = readdir(dir))) {
        if (!strcmp(ent->d_name, ".") || !strcmp(ent->d_name, "..")) {
            continue;
        }

        /* open the plugin */
        if (asprintf(&path, "%s/%s", plugins_dir, ent->d_name) == -1) {
            error_print(0, "asprintf() failed (%s).", strerror(errno));
            rc = -1;
            break;
        }
        /*RTLD_LAZY:暂缓决定，等有需要时再解出符号 
         *以RTLD_LAZY模式打开动态库，返回一个句柄给调用进程，如果失败，则返回。
         */
        handle = dlopen(path, RTLD_LAZY);
        if (!handle) {
            error_print(0, "Opening plugin \"%s\" failed (%s).", path, dlerror());
            free(path);
            rc = -1;
            break;
        }
        free(path);

        /* allocate new plugin */
        /* 分配一个新的plugin空间，并将新分配的men挂载plugins结构下*/
        mem = realloc(*plugins, (*plugin_count + 1) * sizeof **plugins);
        if (!mem) {
            error_print(0, "realloc() failed (%s).", strerror(errno));
            dlclose(handle);
            rc = -1;
            break;
        }
        *plugins = mem;

        /* find required functions */
        /* plugins结构中有两个必须的回调函数，一个是init_cb,另一个是cleanup_cb
         * 通过 void *dlsym(void *handle, const char* symbol);，
         * handle是使用dlopen函数之后返回的句柄，
         * symbol是要求获取的函数的名称。
         * SRP_INIT_CB定义如下：#define SRP_INIT_CB     "sr_plugin_init_cb"
         * SRP_CLEANUP_CB定义下：#define SRP_CLEANUP_CB  "sr_plugin_cleanup_cb"
         * 此两个CB函数，也就是在开发插件中必须实现的两个入口函数，如果不存在，则加载失败。
         */
        *(void **)&(*plugins)[*plugin_count].init_cb = dlsym(handle, SRP_INIT_CB);
        if (!(*plugins)[*plugin_count].init_cb) {
            error_print(0, "Failed to find function \"%s\" in plugin \"%s\".", SRP_INIT_CB, ent->d_name);
            dlclose(handle);
            rc = -1;
            break;
        }

        *(void **)&(*plugins)[*plugin_count].cleanup_cb = dlsym(handle, SRP_CLEANUP_CB);
        if (!(*plugins)[*plugin_count].cleanup_cb) {
            error_print(0, "Failed to find function \"%s\" in plugin \"%s\".", SRP_CLEANUP_CB, ent->d_name);
            dlclose(handle);
            rc = -1;
            break;
        }

        /* finally store the plugin */
        /*最后，本次so解析成功，保存本次so的解析结果，执行一下次目录文件的读取*/
        (*plugins)[*plugin_count].handle = handle;
        (*plugins)[*plugin_count].private_data = NULL;
        ++(*plugin_count);
    }

    closedir(dir);
    return rc;
}
```
### 1.5 init_cb

```c
// srpd_plugin_s 结构中定义了 init 的回调函数
// 如下：
typedef int (*srp_init_cb_t)(sr_session_ctx_t *session, void **private_data);
// 在 load plugin 时，
#define SRP_INIT_CB     "sr_plugin_init_cb"
init_cb = dlsym(handle, SRP_INIT_CB);
// 在 sysrepo-plugind 的 main 实现时，需要对 plugin 的初始化，实际就是需要用户对sr_plugin_init_cb() 实现，是完成该 plugin 的资源分配，用户感兴趣的事情做订阅，Ｍodule change RPC/Action, Notify, get——items 等操作，均在此处完成。有如下例子，请参考。
 
int sr_plugin_init_cb(sr_session_ctx_t *session, void **private_ctx)
{
    int rc;
    struct plugind_ctx *ctx;
    ctx = calloc(1, sizeof *ctx);
    if (!ctx) 
    {
        rc = SR_ERR_NOMEM;
        goto error;
    }
 
    /*在下面初始与之有关的操作，例如，本地数据结构的初始化，sysrepo的订阅初始化*/
    ...
  
    SRP_LOG_DBGMSG("plugin initialized successfully.");
    ctx->session = session;
    *private_ctx = ctx;
    return SR_ERR_OK;
 
error:
    SRP_LOG_ERR("plugin initialization failed (%s).", sr_strerror(rc));
    sr_unsubscribe(ctx->subscription);
    free(ctx);
    return rc;
}
```
### 1.6 cleanup_cb

```c
// srpd_plugin_s 结构中定义了 cleanup 的回调函数
// 如下：
typedef void (*srp_cleanup_cb_t)(sr_session_ctx_t *session, void *private_data);
// 在 load plugin 时，
#define SRP_CLEANUP_CB  "sr_plugin_cleanup_cb"
cleanup_cb = dlsym(handle, SRP_CLEANUP_CB);
// 所以，对于用户来就，是需要对 sr_plugin_cleanup_cb() 实现，回收 plugin 在初始化时分配的资源。例如下面的 cleanup 实现，可以参考
 
void sr_plugin_cleanup_cb(sr_session_ctx_t *session, void *private_ctx)
{
    (void)session;
 
    struct plugind_ctx *ctx = (struct plugind_ctx *)private_ctx;
    sr_unsubscribe(ctx->subscription);
    free(ctx);
    
    nb_terminate();
    yang_terminate();
    
    SRP_LOG_DBGMSG("plugin cleanup finished.");
}
```
## 二、sysrepo plugin动态库示例
以下是本人大概写的一个 动态库的示例：
```c
#include <unistd.h>
#include <signal.h>
#include <inttypes.h>
#include "sysrepo/values.h"
#include "gwttacl.h"
#include <libubox/list.h>
#include <syslog.h>
#include <net/if.h>
#include <sys/ioctl.h>
#include <arpa/inet.h>
#include <pthread.h>
#include <sys/socket.h>
#include <linux/netlink.h>
#include <linux/rtnetlink.h>
#include <net/if.h>
#include <errno.h>
#include <dbus/dbus.h>

#include <gio/gio.h>
#include <glib.h>
#include <glib-object.h>
#include <gio/gdbusconnection.h>


static sr_subscription_ctx_t *acl_subscrip = NULL;

char g_an[MAX_STR_LEN] = {0};
acl_cfg_t *g_acl_val = NULL;
struct list_head acl_cfg_list = LIST_HEAD_INIT(acl_cfg_list);



static void
print_acl_config(struct list_head *list)
{
    acl_cfg_t *acl;
    list_for_each_entry(acl, list, head) {

    }
}



static void
free_acl_cfg_list(struct list_head *node)
{
    acl_cfg_t *p, *tmp;
    list_for_each_entry_safe(p, tmp, node, head) {
        list_del(&p->head);

        if (p) {
            free (p);
        }
    }

    memset(g_an, 0, sizeof(g_an));
}



static int acl_config_get(char *aclname, char *objpath)
{


    return rc;
}



static int
sysrepo_commit_acl(sr_session_ctx_t *sess, struct list_head *list)
{
    char xpath[XPATH_MAX_LEN];
    char name_xpath[128];
    int rc = SR_ERR_OK;
    sr_val_t val = { 0 };
    acl_cfg_t *acl_val;
    char dest_prefix[32] = {0};
    int mask_len = 0;
    char iface[32] = {0};
    
    rc = sr_delete_item(sess, "/XXX-filter-config:filter-configs/filter-config", SR_EDIT_DEFAULT);

    if (SR_ERR_OK != rc) {
        WRN("ACL: Error by sr_set_item: %s", sr_strerror(rc));
    }
    printf("func=%s end, line=%d\n", __FUNCTION__, __LINE__);
    list_for_each_entry(acl_val, list, head) {
        if (strlen(acl_val->name) > 0) {
            val.type = SR_LIST_T;
            val.data.string_val = acl_val->name;
            sprintf(name_xpath, "/XXX-filter-config:filter-configs/filter-config[name='%s']", acl_val->name);
            rc = sr_set_item(sess, xpath, &val, SR_EDIT_DEFAULT);

            if (SR_ERR_OK != rc) {
                WRN("ACL: Error by sr_set_item: %s", sr_strerror(rc));
            }
        } else {
            continue;
        }


  
    }
    rc = sr_apply_changes(sess, 0, 0);

    if (SR_ERR_OK != rc) {
        WRN("ACL: Error by routing commit: %s==%d", sr_strerror(rc), __LINE__);
    }

	DBG_LOG("func %s end", __FUNCTION__);

    free_acl_cfg_list(list);

	
    return rc;

}

static int sr_val_acl_set(const sr_val_t *value, sr_change_oper_t op)
{
    char *v = NULL;
    sr_xpath_ctx_t xpath_ctx;
    char an[MAX_STR_LEN] = {0};
    char net_iface[MAX_STR_LEN] = {0};

    //DBG_LOG("ACL:function %s start.xpath=%s", __FUNCTION__, value->xpath);
    /* route name */
    v = sr_xpath_key_value(value->xpath, "filter-config", "name", &xpath_ctx);
    snprintf(an, MAX_STR_LEN, "%s", v);


    if (v != NULL && 0 != strcmp(an, g_an)) {
        g_acl_val = calloc(1, sizeof(*g_acl_val));

        if (NULL == g_acl_val) {
            ERR_SYSLOG("ACL:calloc memory fail.func=%s,xpath=%s\n",
                __FUNCTION__, value->xpath);
            return SR_ERR_NOMEM;
        }

        list_add(&g_acl_val->head, &acl_cfg_list);
        snprintf(g_acl_val->name, MAX_STR_LEN, "%s", an);
        snprintf(g_an, MAX_STR_LEN, "%s", an);

        g_acl_val->oper  = MOD;
    }

    sr_xpath_recover(&xpath_ctx);


    /* gateway */
    if (sr_xpath_node_name_eq(value->xpath, "name")) {

        g_acl_val->oper  = op;
    } 

    /*接收其他的参数*/

    return SR_ERR_OK;
}

static int sr_set_acl_change(sr_change_oper_t op, sr_val_t *old_val, sr_val_t *new_val)
{
    int rc = SR_ERR_OK;

    switch (op) {
        case SR_OP_CREATED:
            if (NULL != new_val) {
                rc = sr_val_acl_set(new_val, op);
            }

            break;

        case SR_OP_DELETED:
            if (NULL != old_val) {
                rc = sr_val_acl_set(old_val, op);
            }

            break;

        case SR_OP_MODIFIED:
            if (NULL != old_val && NULL != new_val) {
                rc = sr_val_acl_set(new_val, op);
            }

            break;

        case SR_OP_MOVED:
            if (NULL != new_val) {
                DBG("MOVED: %s after %s", new_val->xpath, NULL != old_val ? old_val->xpath : NULL);
            }

            break;
    }

    return rc;
}





static void acl_del_data(acl_cfg_t *acl_cfg)
{

    return ;
}


static void acl_apply()
{
    acl_cfg_t *acl_cfg = NULL;
    print_acl_config(&acl_cfg_list);

    list_for_each_entry(acl_cfg, &acl_cfg_list, head) {
        if (strlen(acl_cfg->name) > 0) {
            if (acl_cfg->oper == DELETE ) {
                DBG_LOG("ACL: delete route : %s\n", acl_cfg->name);
                //处理删除操作
                acl_del_data(acl_cfg);
            } else {
                //处理添加、修改操作
                acl_set_dbus_data(acl_cfg);
            }
        }
    }

    free_acl_cfg_list(&acl_cfg_list);

    DBG_LOG("ACL:function %s end.", __FUNCTION__);
    return;
}

static int acl_to_model(sr_session_ctx_t *sess, const char *xpath)
{
    sr_change_iter_t *it = NULL;
    sr_change_oper_t oper;
    sr_val_t *old_value = NULL;
    sr_val_t *new_value = NULL;
    int rc = SR_ERR_OK;
    int ret = SR_ERR_OK;
    char change_path[XPATH_MAX_LEN] = {0,};

    snprintf(change_path, XPATH_MAX_LEN, "%s//.", xpath);
    rc = sr_get_changes_iter(sess, change_path, &it);

    if (SR_ERR_OK != rc) {
        ERR("ACL: get changes iter fail, line=%d, xpath=%s, rc=%s\n",
            __LINE__, xpath, sr_strerror(rc));
        sr_free_change_iter(it);
        return rc;
    }

    while (SR_ERR_OK == (ret = sr_get_change_next(sess, it,
                    &oper, &old_value, &new_value))) {
        rc = sr_set_acl_change(oper, old_value, new_value);
        sr_free_val(old_value);
        sr_free_val(new_value);

        if (SR_ERR_OK != rc) {
            sr_free_change_iter(it);
            memset(g_an, 0, MAX_STR_LEN);
            free_acl_cfg_list(&acl_cfg_list);
            return rc;
        }
    }

    memset(g_an, 0, MAX_STR_LEN);
    sr_free_change_iter(it);
    DBG_LOG("ACL : get all changes successfully, xpath=%s\n", xpath);
    return SR_ERR_OK;
}

static int acl_change_cb(sr_session_ctx_t *session,
    const char *module_name,
    const char *xpath,
    sr_event_t event,
    uint32_t request_id,
    void *private_ctx)
{
    if (SR_EV_CHANGE == event) {
        return acl_to_model(session, xpath);
    }

    if (SR_EV_DONE == event) {
        acl_apply();
    }

    return SR_ERR_OK;
}


static int acl_db_init(sr_session_ctx_t *session)
{
    int rc = SR_ERR_OK;
    //get 所有的配置
    if (SR_ERR_OK != (rc = acl_config_get(NULL, NULL))) {
        return rc;
    }

    //print_acl_config(&acl_cfg_list);
    //同步到running库
    return sysrepo_commit_acl(session, &acl_cfg_list);
}



int sr_plugin_init_cb(sr_session_ctx_t *session, void **private_ctx)
{
    char xpath[XPATH_MAX_LEN];
    sr_val_t val = { 0, };
    int rc = SR_ERR_OK;

    /*get 所有的相关配置，同步到 running库中*/
    if (SR_ERR_OK != (rc = acl_db_init(session))) {
        return rc;
    }

    rc = sr_module_change_subscribe(session, "XXX-filter-config",
            "/XXX-filter-config:filter-configs/filter-config",
            acl_change_cb, *private_ctx, 0, SR_SUBSCR_DEFAULT,
            &acl_subscrip);

    if (SR_ERR_OK != rc) {
        ERR_MSG("ACL: gwtt-filter-config subscribe fail\n");
        goto error;
    }

    return SR_ERR_OK;

error:

    if (acl_subscrip) {
        sr_unsubscribe(acl_subscrip);
    }

    return rc;

}

void sr_plugin_cleanup_cb(sr_session_ctx_t *session, void *private_ctx)
{
    if (acl_subscrip) {
        sr_unsubscribe(acl_subscrip);
    }

    SRP_LOG_DBGMSG("Plugin gwtt_route cleaned-up end successfully");
    return;
}



```
