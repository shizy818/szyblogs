---
title:  “InnoDB 内核阅读笔记（五）- 存储管理”
mathjax: true
layout: post
date:   2018-05-10 08:00:12 +0800
categories: database
---

InnoDB存储引擎最小的存储单位是页（Page），默认16KB大小；在页的基础上逻辑的分为区（Extent），段（Segment）和表空间（Tablespace）。InnoDB引擎对于空间的申请是以区为单位（1MB），总共64页，提高空间申请效率的同时，使得数据可以根据键值顺序存放。

在物理磁盘上存在扇区（Sector），机械硬盘通常为512字节，固态硬盘为4KB；文件系统将扇区映射为块（Block），通常大小为4KB。对应关系为下图（1 page = 4 block = 32 sector）：  
![image01]({{site.baseurl}}/image/20180510/sector_block_page.png)


# **fsp & fil & os文件夹**

文件名称 |  说明
:----|:----
fsp0fsp.cc | 物理文件结构与实现
fil0fil.cc | 表空间内存数据结构
os0file.cc | 底层文件操作

# **物理存储**

## 页
InnoDB页包含38个字节的头部和8个字节尾部。

名称 | 大小 |  说明
:----|:----|:----
FIL_PAGE_SPACE_OR_CHKSUM | 4 | 验证信息
FIL_PAGE_OFFSET | 4 | 页在表空间中的偏移量
FIL_PAGE_PREV | 4 | 前一个页的偏移量
FIL_PAGE_NEXT | 4 | 后一个页的偏移量
FIL_PAGE_LSN | 8 | 页最后被修改的LSN
FIL_PAGE_TYPE | 2 | 页的类型
FIL_PAGE_FILE_FLUSH_LSN | 8 | 在系统表空间中使用，判断数据库是否正常关闭
FIL_PAGE_SPACE_ID | 4 | 表空间ID
FIL_PAGE_DATA | | 开始存储数据
FIL_PAGE_END_LSN_OLD_CHKSUM | 8 | 低位4个字节存储页验证信息，高位4字节等于FIL_PAGE_LSN最后4个字节

## 区

区的申请由space header（空间头部信息）管理，如下图所示。space header中区分为区（extent），保存在FSP_FREE链表中；碎片区（frag extent），保存在FSP_FREE_FRAG和FSP_FULL_FRAG链表中。函数fsp_alloc_free_extent -> fsp_fill_free_list用于申请区。

名称 | 大小 |  说明
:----|:----|:----
FSP_SPACE_ID | 4 | 表空间ID
FSP_NOT_USED | 4 | 未使用
FSP_SIZE | 4 | 当前已有页的总大小
FSP_FREE_LIMIT | 4 | 当前已经用到位置，大于此位置的部分表示还未被初始化
FSP_SPACE_FLAG | 4 | REDUNDANT/COMPACT/COMPRESSED/DYNAMIC
FSP_FRAG_N_USED | 4 | 碎片区中已经使用页的总数
FSP_FREE | 16 | 空闲区链表
FSP_FREE_FRAG | 16 | 空闲碎片区链表
FSP_FULL_FRAG | 16 | 已经完全使用的碎片区链表
FSP_SEG_ID | 8 | 下一个段ID
FSP_SEG_INODES_FULL | 16 | 已经使用的inode页链表
FSP_SEG_INODES_FREE | 16 | 空闲inode页链表 

区中64个页的使用情况，通过位图在区描述符（extent descriptor）追踪。区描述符在页中的偏移是150字节处（38字节头 + 112字节space header）。

名称 | 大小 |  说明
:----|:----|:----
XDES_ID | 8 | 区所属段ID
XDES_FLST_NODE | 12 | 该区所在链表（FSP_FREE / FSP_FRAG_FREE / FSP_FRAG_FULL）中的位置
XDES_STATE | 4 | 区的状态 （XDES_FREE / XDES_FREE_FRAG /XDES_FULL_FRAG / XDES_FSEG）
XDES_BITMAP | 16 | 每个页用2位表示 (XDES_FREE_BIT / XDES_CLEAN_BIT)

InnoDB定义了一个页保存256个区描述符，所以区描述符所在页的偏移量都是16384(256 * 64)的倍数。第0页包含有space header信息，后面的区描述符页仅保留该空间。  
![image02]({{site.baseurl}}/image/20180510/page_extent.png)

## 段

段用来保存特定类型的数据，比如聚集索引的叶子节点段（leaf segment）和非叶子节点段（non leaf segment）。为了节省空间，每个段设计了32个碎片页，段中的空间首先保存在这32个页中，超出容量后再以区的方式申请空间。碎片页的申请函数为fsp_alloc_free_page。碎片区保存在space header中，不分配给段。

segment inode用于保存段信息，入下表所示。为了找到segment inode所在位置，还需要一个segment header。对于用户表来说，segment header总是保存在root页中，分别指向叶子节点segment inode和非叶子节点segment inode。函数fseg_create_general创建一个段，初始化segment inode以及segment header.

名称 | 大小 |  说明
:----|:----|:----
FSEG_ID | 8 | 段ID
FSEG_NOT_FULL_N_USED | 4 | 链表FSEG_NOT_FULL中所有已经分配页的数量
FSEG_FREE | 16 | 空闲区链表
FSEG_NOT_FULL | 16 | 全部页未被使用的区链表
FSEG_FULL | 16 | 全部页已被使用的区链表
FSEG_MAGIC_N | 4 |
FSEG_FRAG_ADD | 128 | 碎片页链表，一共32页

下图是表空间，段，区，页和行记录整体逻辑关系:  
![image03]({{site.baseurl}}/image/20180510/ts_seg_ext_page.png)

# **内存数据结构**

InnoDB存储引擎对于文件管理通过数据结构fil_system_t， fil_space_t和fil_node_t完成。

fil_system_t是文件系统的逻辑结构，全局只有一个文件系统逻辑结构对象(fil_system)，统一管理所有文件节点的各种操作。

变量 | 类型 |  说明
:----|:----|:----
mutex | ib_mutex_t | 保护文件系统逻辑结构的互斥锁，对下列成员变量的独占访问
spaces | hash_table_t* | 文件空间的哈希索引表（依赖文件空间ID），快速访问文件空间
name_hash | hash_table_t* | 文件空间的哈希索引（依赖文件名）
LRU | UT_LIST_BASE_NODE_T(fil_node_t) | 文件系统最近打开过的文件节点链表
unflushed_spaces | UT_LIST_BASE_NODE_T(fil_space_t) | 包含未刷新更新的文件空间链表
n_open | ulint | 当前已打开文件节点个数
max_n_open | ulint | 文件系统中最大可以打开的文件节点个数
modification_counter | ib_int64_t | 修改文件节点次数
max_assigned_id | ulint | 已有文件空间最大ID
tablespace_version | ib_int64_t | 
space_list | UT_LIST_BASE_NODE_T(fil_space_t) | 文件空间的链表，实现对文件空间的遍历访问

fil_space_t是文件空间的逻辑结构，文件空间指由若干文件节点组成的一个逻辑文件。InnoDB定义了两种文件空间类型：
- FIL_TABLESPACE： 默认文件节点名称是idbdata*
- FIL_LOG: 默认文件节点名称是ib_logfile*

变量 | 类型 |  说明
:----|:----|:----
name | char* | 该文件空间第一个文件节点的文件名
id | ulint | 文件空间id号
tablespace_version | ib_int64_t | 文件空间版本号
mark | ibool |
stop_ios | ibool |
stop_new_ops | ibool |
purpose | ulint | 文件空间类型
chain | UT_LIST_BASE_NODE_T(fil_node_t) | 此文件空间中包含的文件节点链表
size | ulint | 文件空间所有文件节点包含页的个数
n_reserved_extents | ulint | 为表空间的操作预留的空闲区个数，比如B+树的节点分裂会导致表空间的size增加，为了防止增加后超过表空间size最大值，表空间预留这么多个区
n_pending_flushes | ulint | 该表空间等待刷新操作个数
n_pending_ops | ulint | 该表空间等待操作（合并插入缓存，锁验证等）个数
hash | hash_node_t | 哈希链表
name_hash | hash_node_t | 名字哈希链表
latch | rw_lock_t | 对表空间并发操作进行保护
unflushed_spaces | UT_LIST_NODE_T(fil_space_t) | 未刷新空间链表
is_in_unflushed_spaces | bool |  此文件空间是否在unflushed_spaces链表中
space_list | UT_LIST_NODE_T(fil_space_t) | 用于将此文件空间连接到文件系统的space_list上的链表节点

fil_node_t是文件节点的数据结构，便于对文件进行读/写操作。一个文件节点必定在一个文件空间的chain链表上。当
一个文件节点的操作完成时，该文件节点不会立即被关闭，而是被加入到文件系统的LRU链表中。

变量 | 类型 |  说明
:----|:----|:----
name | char* | 文件节点名称
open | ibool | 文件节点是否已经被打开
handle | pfs_os_file_t | 文件节点打开后的文件描述字
sync_event | os_event_t | 组提交同步fsync事件
is_raw_disk | ibool | 文件是否是裸设备
size | ulint | 文件节点包含页的个数
n_pending | ulint | 该文件节点上等待读/写操作的个数
n_pending_flushes | 该文件节点上等待刷新操作个数
being_extended | ibool | 文件节点是否正在扩展
modification_counter | ib_int64_t | 文件节点被更新次数
flush_counter | ib_int64_t | 文件节点被刷新次数
chain | UT_LIST_NODE_T(fil_node_t) | 将该文件节点连接到文件空间变量fil_space_t中chain上的链表节点
LRU | UT_LIST_NODE_T(fil_node_t) | 将该文件节点连接到文件系统变量fil_system_t中lru上的链表节点

# **异步I/O**

InnoDB存储引擎共有4个异步I/O线程，分别处理异步读I/O，异步写I/O，插入缓存I/O，以及重做日志I/O。每个异步I/O
由数据结构os_aio_array_t表示，变量n_segments表示每个类型的异步I/O线程数量，5.6默认值值是4。

变量 | 类型 |  说明
:----|:----|:----
mutex | os_ib_mutex_t | 对异步I/O请求队列的并发访问进行保护的互斥量
not_full | os_event_t | 异步I/O请求队列中是否还有空闲slot存放新的I/O请求事件
is_empty | os_event_t | I/O请求队列中是否为空的事件
n_slots | ulint | 异步I/O请求队列中总共可以存放的I/O请求数量
n_segments | ulint | 异步I/O请求队列被分成段的个数，每个段有一个I/O处理现场
cur_seg | ulint | 服务下一次I/O请求的段
n_reserved | ulint | 异步I/O请求队列中已经存放的I/O请求的数量
slots | os_aio_slot_t* | 存放I/O请求的数组空间

os_aio_array_t由函数`os_aio_init -> os_aio_array_create`初始化，如果定义了LINUX_NATIVE_AIO，使用操作系统内核支持
的异步I/O，代码如下：
```cpp
	/* Initialize the io_context array. One io_context
	per segment in the array. */

	array->aio_ctx = static_cast<io_context**>(
		ut_malloc(n_segments * sizeof(*array->aio_ctx)));

	for (ulint i = 0; i < n_segments; ++i) {
		if (!os_aio_linux_create_io_ctx(n/n_segments,
						&array->aio_ctx[i])) {
			/* If something bad happened during aio setup
			we should call it a day and return right away.
			We don't care about any leaks because a failure
			to initialize the io subsystem means that the
			server (or atleast the innodb storage engine)
			is not going to startup. */
			return(NULL);
		}
	}

	/* Initialize the event array. One event per slot. */
	io_event = static_cast<struct io_event*>(
		ut_malloc(n * sizeof(*io_event)));

	memset(io_event, 0x0, sizeof(*io_event) * n);
	array->aio_events = io_event;
```

InnoDB存储引擎启动时调用`innobase_start_or_create_for_mysql -> os_thread_create`创建异步I/O线程；异步I/O线程不断执行
`fil_aio_wait -> os_aio_linux_handle -> os_aio_linux_collect -> io_getevents`，判断是否异步线程队列的slot中是否有
已经完成的I/O操作。如果操作系统不支持原生异步I/O，则通过函数os_aio_simulated_handle来模拟异步I/O实现。