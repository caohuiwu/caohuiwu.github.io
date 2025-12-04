---
title: 《mysql-源码分析》Innodb启动
date: 2020-08-11
categories:
  - [mysql]
tags:
  - [mysql源码]
---

<style>
.orange {
   color: orange
}
.red {
   color: red
}
code {
   color: #0ABF5B;
}
</style>

    这是“mysql”系列的第四篇文章，主要介绍的是innodb启动源码。

# 一、mysql

<code>MySQL</code> 是一种广泛使用的开源关系型数据库管理系统（RDBMS--Relational Database Management System）

<!-- more -->

之前的文章讲解了`MySQL Server`的启动过程，现在深入解析以下`Innodb`的启动过程源码。

# 二、Innodb 启动
回顾一下`MySQL Server`的启动过程，执行`init_server_components()`完成组件的初始化。
```text
mysqld_main() 
  -> init_server_components() 
    -> plugin_register_builtin_and_init_core_se()
      -> plugin_initialize()
        -> plugin->plugin->init(plugin)
            -> innobase.init()
```
MySQL的存储引擎（如Innodb）是以插件的形式注册到服务器中。`init_server_components()`的核心作用是加载并初始化这些插件。关键代码如下：
```cpp
// 文件：sql/sql_plugin.cc
if (plugin_register_builtin_and_init_core_se(...)) {
  // 如果初始化失败，终止启动
  unireg_abort(MYSQLD_ABORT_EXIT);
}
```
进入`plugin_register_builtin_and_init_core_se`代码内：
```cpp
bool plugin_register_builtin_and_init_core_se(int *argc, char **argv)
{
  for (struct st_mysql_plugin **builtins= mysql_mandatory_plugins;
       *builtins || mandatory; builtins++)
  {
    plugin_initialize(plugin_ptr))
  }

```
继续进入`plugin_initialize`
```cpp
static int plugin_initialize(st_plugin_int *plugin)
{
  plugin->plugin->init(plugin)
}
```
遍历内置插件列表（`mysql_mandatory_plugins`），包括 InnoDB。对每个插件调用其`init()`函数，完成初始化。

## 2.1、innodb 插件
MySQL的**插件**通过 `mysql_declare_plugin` 定义，包含插件名称、类型、`init/deinit`函数等。例如 `Innodb` 的插件定义如下：
```cpp
// storage/innobase/handler/ha_innodb.cc
mysql_declare_plugin(innobase) {
    MYSQL_STORAGE_ENGINE_PLUGIN,
    &innobase_hton,                     // 插件接口结构体
    innobase_init,                      // 插件初始化函数
    innobase_deinit,                    // 插件销毁函数
    "InnoDB",
    "Oracle Corporation",
    PLUGIN_LICENSE_GPL,
    innodb_plugin_status,
    0x0001,
    NULL,
    NULL,
    NULL
} mysql_declare_plugin_end;
```
从InnoDB插件的定义来看，其初始化函数为 `innobase_init`，位于`handler/ha_innodb.cc` 文件中，这个函数只有一个参数，该参数是`handlerton`结构体类型的指针。

`handlerton` 可以理解为`MySQL server`层与`engine`层的一个纽带，一个存储引擎对应一个该类型的实例，用来提供在全局范围内访问存储引擎的功能。`handlerton` 结构体内部定义了很多接口，比如`commit, rollback, create , drop_database` 等等，在存储引擎初始化阶段，会把这些接口赋值为存储引擎实际的实现，以便在`server`层能够调用存储引擎的接口。`handlerton` 结构体定义在 `sql/handler.h`文件中。


### 2.1.1、init()
```cpp
static int innobase_init(void *p) {
    handlerton *innobase_hton = (handlerton *)p;
    // 设置 InnoDB 接口（事务、锁等）
    if (innodb_init()) {  // 调用 InnoDB 核心初始化
        return 1;
    }
    return 0;
}
```
`innobase_init`函数大约680行代码，是InnoDB的初始化函数，具体做了哪些事情？总结如下：
- 初始化`innobase_hton`，它是一个`handlerton`类型的指针，在存储引擎初始化阶段，会把这些接口赋值为`InnoDB`存储引擎实际的实现，以便在`server`层能够调用存储引擎的接口。
- InnoDB相关参数的检查和初始化，包括共享表空间、临时表空间、`undo`表空间、`redo`日志文件、`double write`文件等等。
- 调用 `innobase_start_or_create_for_mysql` 函数，这个函数非常重要，后面详细分析这个函数。

#### 2.1.1.1、innobase_start_or_create_for_mysql 函数
这个函数位于源码文件`storage/innobase/srv/srv0start.cc`，主要作用是启动innodb引擎
```cpp
dberr_t
innobase_start_or_create_for_mysql(void)
/*====================================*/
{
  //调用 srv_boot 函数，启动innodb server，实际上其内部调用了很多初始化的函数，可以理解为相关参数和组件的初始化。
  srv_boot();
  // 调用log_init函数，初始化redo log系统。
  log_init();
  //创建innodb主线程，线程函数为srv_master_thread。
  os_thread_create(
      srv_master_thread,
      NULL, thread_ids + (1 + SRV_MAX_N_IO_THREADS));
  //创建purge系统协调线程，线程函数为srv_purge_coordinator_thread。
  os_thread_create(
    srv_purge_coordinator_thread,
    NULL, thread_ids + 5 + SRV_MAX_N_IO_THREADS);
              
  // 创建purge系统工作线程，数量由srv_n_purge_threads参数决定，线程函数为srv_worker_thread。
  for (i = 1; i < srv_n_purge_threads; ++i) {
    os_thread_create(
      srv_worker_thread, NULL,
      thread_ids + 5 + i + SRV_MAX_N_IO_THREADS);
  }

}
```
从以上的分析可以看出，`innobase_start_or_create_for_mysql`函数做了很多事情，调用了很多函数，创建了很多线程，我们没有细究每个函数、每个线程都有什么作用，但是这不影响对InnoDB整个初始化过程的了解。后面将会针对InnoDB的具体模块，从源码角度分析其具体的实现。

最后附上InnoDB最大线程数量的计算方法，如下：
```cpp
srv_max_n_threads = 1   /* io_ibuf_thread */
                     + 1 /* io_log_thread */
                     + 1 /* lock_wait_timeout_thread */
                     + 1 /* srv_error_monitor_thread */
                     + 1 /* srv_monitor_thread */
                     + 1 /* srv_master_thread */
                     + 1 /* srv_redo_log_follow_thread */
                     + 1 /* srv_purge_coordinator_thread */
                     + 1 /* buf_dump_thread */
                     + 1 /* dict_stats_thread */
                     + 1 /* fts_optimize_thread */
                     + 1 /* trx_rollback_or_clean_all_recovered */
                     + 128 /* added as margin, for use of
                         InnoDB Memcached etc. */
                     + max_connections
                     + srv_n_read_io_threads
                     + srv_n_write_io_threads
                     + srv_n_purge_threads
                     + srv_n_page_cleaners
                     /* FTS Parallel Sort */
                     + fts_sort_pll_degree * FTS_NUM_AUX_INDEX
                       * max_connections;
```

    
参考文章：
[MySQL启动过程详解二：核心模块启动 init_server_components()](https://www.cnblogs.com/juanmaofeifei/p/16111523.html)
[MySQL启动过程详解三：Innodb存储引擎的启动](https://www.cnblogs.com/juanmaofeifei/p/16129144.html)
[MySQL InnoDB存储引擎启动过程源码分析](http://www.mytecdb.com/blogDetail.php?id=91)