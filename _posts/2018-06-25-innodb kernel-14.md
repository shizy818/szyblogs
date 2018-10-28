---
title:  “InnoDB 内核阅读笔记（十四）- 服务管理”
date:   2018-06-25 08:00:12 +0800
categories: database
---

InnoDB初始化的时候调用innobase_start_or_create_for_mysql来启动存储引擎：
- 参数和内存初始化
- 文件的创建加载
- 相关数据的恢复
- 相关事务的回滚
- 工作线程的创建

系统参数包括InnoDB引擎的配置参数，系统表空间等；申请内存包括用于缓存页的buffer pool和用于缓存重做日志的log buffer。首次启动时需要创建全局表空间文件和保存重做日志的日志文件。对于非首次启动，需要加载相关的数据和日志文件，判断是否需要对数据进行恢复（Recovery）或者回滚（Rollback）操作。除此之外，还需要创建几种核心的后台线程，用于实现异步I/O读写，事务锁超时监控，数据脏页的刷新等。


# **srv文件夹**

文件名称 | 说明
:----|:----
srv0connc.cc | 并发管理模块
srv0mon.cc | 监控模块
srv0srv.cc | 服务管理
srv0start.cc | 启动服务

# **InnoDB初始化**

## 相关参数

1. srv_unix_file_flush_method， 指定数据文件的刷盘方式
    - SRV_UNIX_FSYNC（默认）
    
        所有写入都进行fsync，确保数据和数据文件属性都写到磁盘
    - SRV_UNIX_O_DSYNC
    
        日志文件以O_DSYNC打开，只确保数据写到磁盘
    - SRV_UNIX_LITTLESYNC
    
        数据文件不做fsync，日志文件做fsync
    - SRV_UNIX_NOSYNC
        
        所有的写入都不进行fsync
    - SRV_UNIX_O_DIRECT
    
        不经过操作系统缓存直接把数据写到磁盘，有利于大量随机写入，但是会降低顺序读写的效率

2. srv_n_read_io_threads/srv_n_write_io_threads，通过参数innodb_read_io_threads和innodb_write_io_threads指定读写异步I/O数量，默认值为4。

srv_sys_t结构

变量 | 类型 | 说明
:----|:----|:----
tasks_mutex | ib_mutex_t | 保护任务队列互斥量
tasks | UT_LIST_BASE_NODE_T(que_thr_t) | 任务队列
mutex | ib_mutex_t | 保护线程相关互斥量
n_sys_threads | ulint | 线程数量
sys_threads | srv_slot_t* | 保存的线程，用于暂停与恢复等相关管理操作
n_threads_active[SRV_MASTER + 1] | ulint | 活跃线程数

srv_slot_t结构

变量 | 类型 | 说明
:----|:----|:----
type | srv_thread_type | 线程类型，有效值SRV_WORKER，SRV_PURGE，SRV_MASTER
in_use | ibool | 是否使用
suspended | ibool | 线程是否被挂起
suspend_time | ib_time_t | 线程被挂起时间，用于锁超时判断
wait_timeout | ulong | 超时等待时间
event | os_event_t | 用于线程挂起事件
thr | que_thr_t* | 被挂起的查询线程

srv_conc_slot_t结构

变量 | 类型 | 说明
:----|:----|:----
event | os_event_t | 等待的事件
reserved | ibool | 如果slot已分配，则为TRUE
wait_ended | ibool |
srv_conc_queue | srv_conc_node_t | 等待队列

对于每个数据库会话，都会建立对应的线程对象srv_slot_t。大量的并发用户会话会产生竞争。srv_conc_slot_t用于线程的并发控制管理。参数innodb_thread_concurrency设置最大的用户并发线程，超过该阈值则发生等待操作。

## 启动后台线程

InnoDB启动的时候，需要启动集中不同的后台线程：
- io_handler_thread

    用于异步I/O操作，包括数据页读/写和IBUF操作。io_handler_thread实现的主要是调用fil_aio_wait函数循环监控相应队列。
- srv_lock_timeout_thread/srv_monitor_thread

    打印监控信息和检测事务持有锁。当线程被锁住超过一定时间后，通过该线程来释放锁，从而避免死锁。
- srv_error_monitor_thread

    检测系统中长时间等待的信号，当等待时间过长时，打印错误信息
- srv_master_thread

    负责很多后台任务，比如刷新脏页，合并Insert Buffer，回收undo log等等

# **关闭InnoDB存储引擎**

调用函数innobase_shutdown_for_mysql完成关闭操作。关闭的第一部分是数据持久化，确保数据和日志信息都写入磁盘，然后创建检查点（checkpoint）。当检查点创建完毕后，把最近检查点LSN写入全局表空间的第一页页头，这样重新启动时，可以判断是否需要进行恢复。参数innodb_fast_shutdown默认值是ON，表示仅将重做日志以及脏页刷新到磁盘；若设置为OFF，表示除此之外还需等待Insert Buffer合并操作，以及undo记录的purge操作完成，而这些操作可能会非常耗时。

关闭的第二部分是释放资源。除了缓冲池的内存申请，其他内存申请（redo日志缓存，恢复缓存等）都通过函数ut_malloc完成，并加入到一个列表中。关闭InnoDB时，调用函数ut_free_all_mem遍历ut_mem_block_list，对所有申请的内存进行释放。

# **Master线程**

master线程主要负责以下几项功能：
- 定期将数据写入磁盘
- Insert Buffer中记录合并
- 检查点创建
- Undo数据页回收

参数innodb_flush_log_at_trx_commit用于配置何时把redo log写入到磁盘。该值为0时，表示
每隔1秒钟将log buffer中的数据写入到文件，同时通知文件系统进行文件同步的flush操作，保证数据写入到磁盘。该值为1时（默认），表示每次事务提交都将log buffer中的数据写入文件并通知文件系统同步文件。该值为2时，表示每次事务提交的时候将数据写入日志文件，但是这里的写入仅仅是调用了文件系统的文件写入操作；而文件系统都是有缓存机制的，所以并不能保证日志内容持久化到物理磁盘；每隔1秒钟通知文件系统同步文件。

每秒一次的操作有：
- 日志缓冲刷新到磁盘（总是)
- 合并至多5个插入缓冲（可能，前1秒I/O次数小于5次)
- 刷新100个缓冲池脏页到磁盘（可能，缓冲池中脏页比例超过阈值90%）

每10秒一次的操作有：
- 刷新100个缓冲池脏页到磁盘（可能，前10秒I/O次数小于200次）
- 日志缓冲刷新到磁盘（总是)
- 合并至多5个插入缓冲（总是）
- 删除无用的Undo页（总是）
- 刷新脏页到磁盘（总是，缓冲池中脏页比例超过70%，刷新100个脏页，否则刷新10个脏页）
- 产生一个检查点（总是）

如果InnoDB没有活跃用户，master线程切入background_loop状态：
- 删除无用Undo页（总是）
- 合并20个插入缓冲（总是）
- 不断刷新100个脏页，直到脏页比例低于阈值90%（可能，空闲时跳转到flush_loop)

参考：  
[何登成：innodb_flush_method与File I/O](http://www.orczhou.com/index.php/2009/08/innodb_flush_method-file-io/)