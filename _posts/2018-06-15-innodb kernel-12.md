---
title:  “InnoDB 内核阅读笔记（十二）- 事务处理”
date:   2018-06-15 08:00:12 +0800
categories: database
---

# **trx文件夹**

文件名称 | 说明
:----|:----
trx0purge.cc | purge数据结构及相关操作
trx0rec.cc | undo log定义与相关操作
trx0roll.cc | 事务回滚实现
trx0rseg.cc | 回滚段的实现
trx0sys.cc | 事务系统段的实现
trx0trx.cc | 事务上层操作
trx0undo.cc | undo log段的实现


# **事务系统结构**

事务系统段仅有一个页，用来存储4部分信息：
- 事务相关信息
- 回顾段的segment header信息
- MySQL二进制日志位置信息
- Double Write段信息

TRX_SYS_TRX_ID_STORE用以保存系统中最大事务ID，其更新是在事务每次开始时。TRX_SYS_FSEG_HEADER保存事务系统段的segment header。然后是回滚段信息，一共有128个回滚段（TRX_SYS_N_RSEGS）；每个回滚段占8字节，分别保存回滚段对象的space和offset信息。接着TRX_SYS_MYSQL_LOG_INFO表示事务提交时MySQL数据库上层的二进制日志位置信息，用于保证MySQL数据库和InnoDB存储引擎事务一致性。页最后的200字节开始处（TRX_SYS_DOUBLEWRITE）保存doublewrite段的信息。

内存中保存事务相关信息，主要由数据结构trx_sys_t和trx_t完成。数据结构trx_sys_t表示事务系统的信息，数据结构trx_t描述每一个事务对象。在InnoDB存储引擎中，有一个全局的trx_sys_t对象trx_sys保存事务系统信息。

trx_sys_t结构

变量 | 类型 | 说明
:----|:----|:----
mutex | ib_mutex_t | 并发保护
max_trx_id | trx_id_t | 当前最大事务ID
rw_trx_list | trx_list_t | 当前活跃读写事务链表
ro_trx_list | trx_list_t | 当前活跃只读事务链表
mysql_trx_list| trx_list_t | 内部事务链表
rseg_array[TRX_SYS_N_RSEGS] | trx_rseg_t* | 回滚段对象数组
rseg_history_len | ulint | history链表长度
view_list | UT_LIST_BASE_NODE_T(read_view_t) | read_view视图链表

trx_t结构，整体而言包括事务相关信息、锁、undo日志和read_view。

变量 | 类型 | 说明
:----|:----|:----
mutex | ib_mutex_t | 并发保护
state | trx_state_t | 事务状态，有效值为TRX_STATE_NOT_STARTED，TRX_STATE_ACTIVE，TRX_STATE_PREPARED或者TRX_STATE_COMMITTED_IN_MEMORY
lock | trx_lock_t | 事务锁
is_recovered | ulint | 是否处于恢复状态
op_info | char* | 事务简单文字说明
dict_operation | trx_dict_op_t | 事务是否DDL操作
id | trx_id_t | 事务ID
no | trx_id_t | 事务进入TRX_STATE_COMMITTED_IN_MEMORY状态时的最大事务ID
commit_lsn | lsn_t | 事务提交lsn
table_id | table_id_t | 进行DDL操作表的ID
mysql_log_file_name | const char* |二进制日志文件名
mysql_log_offset | ib_int64_t | 二进制日志写入的偏移量
mysql_thd | THD* | 事务线程
trx_list | UT_LIST_NODE_T(trx_t) | 事务链表
mysql_trx_list | UT_LIST_NODE_T(trx_t) | 内部事务链表
undo_mutex | ib_mutex_t |  保护undo操作的互斥量
undo_no | undo_no_t | 下一条undo log记录的ID
last_sql_stat_start | trx_savept_t | 保存点，可用于回滚
rseg | trx_rseg_t* | 分配给事务的回滚段
insert_undo | trx_undo_t* | insert undo日志
update_undo | trx_undo_t* | update undo日志
roll_limit | undo_no_t | 回滚到的undo日志
pages_undone | ulint | 回滚使用undo页的数量
undo_no_arr | trx_undo_arr_t* | 回滚时创建的槽，保存回滚的undo日志的undo no
error_state | dberr_t | 错误代码，0表示没有错误
error_info | const dict_index_t* | 如果出现重复键值错误，该指针指向索引
sess | sess_t* | 事务会话
read_view | read_view_t* | 事务持有的read view对象链表 

trx_lock_t结构

变量 | 类型 | 说明
:----|:----|:----
que_state | trx_que_t | 线程运行状态，有效值TRX_STATE_ACTIVE，TRX_QUE_RUNNING，TRX_QUE_LOCK_WAIT等
wait_lock | lock_t* | 事务等待的锁
deadlock_mark | ib_uint64_t | 死锁标记
lock_heap | mem_heap_t* | 锁分配的内存堆
trx_locks | UT_LIST_BASE_NODE_T(lock_t) | 事务持有锁的链表
table_locks | ib_vector_t* | 事务持有的表锁
cancel | ibool | 事务是否回滚

# **double write段**

页刷新会遇到部分写（partial write）问题。InnoDB存储引擎的页的大小是16KB，但是操作系统仅仅保证512字节是的写入是原子的。为了避免部分写造成的问题，页刷新到磁盘时首先写入到doublewrite buffer中，当收集满后强制刷新到表空间中的doublewrite段中，然后再通过buf_dblwr_flush_buffered_writes将缓冲池中的页刷新到磁盘。若发生部分写问题，可以通过doublewrite中的页进行恢复。

![image01]({{site.baseurl}}/image/20180615/doublewrite.png)

doublewrite段的segment header保存在事务系统段的TRX_SYS_DOUBLEWRITE_FSEG。函数buf_dblwr_create用于初始化doublewrite段，由于doublewrite的写入是顺序的，而段的分配首先申请32个碎片页，所以一共需要申请128+32页（2 * TRX_SYS_DOUBLEWRITE_BLOCK_SIZE + FSP_EXTENT_SIZE / 2），32个碎片页被浪费掉了。

doublewrite对象在物理上有2MB的存储空间，在内存中也有对应的数据结构buf_dblwr_t，并且存在全局对象buf_dblwr。

buf_dblwr_t结构

变量 | 类型 | 说明
:----|:----|:----
mutex | ib_mutex_t | 互斥量，将页批量刷新到doublewrite时需要持有该对象
block1 | ulint | doublewrite对象中第1个区开始的page页号
block2 | ulint | doublewrite对象中第2个区开始的page页号
first_free | ulint | 下一个可写入的doublewrite buffer的位置；128表示已满
b_event | os_event_t | 线程等待批量刷新事件
s_event | os_event_t | 线程等待单页刷新事件
write_buf | bytes* | doublewrite buffer
write_buf_unaligned | bytes* | 未对齐的doublewrite buffer
buf_block_arr | buf_page_t** | 刷新页列表

# **undo日志**

undo日志有两个作用：
- 实现事务的原子性，当事务由于意外情况未能提交时，可以使得事务回滚
- 实现一致性非锁定读，通过多版本控制的方式读取行的一个快照数据

InnoDB引擎中，undo日志通过两个对象存放：回滚段与undo段。回滚段保存undo段segment header所在页的位置，一个回滚段可以保存1024个undo段信息，理论上可以支持128*1024个并发事务。事务系统端，回滚段与undo段的关系：

![image02]({{site.baseurl}}/image/20180615/rollback_undo.png)

回滚段中有一个history链表，根据事务的提交顺序逆序存放undo日志。当进行purge操作时，从尾部开始读取undo日志，判断是否可以回收空间。

回滚段的内存数据结构为trx_rseg_t。回滚段保存undo段在表空间的位置，并将已经提交事务的undo日志保存到history链表中。

变量 | 类型 | 说明
:----|:----|:----
id | ulint | 回滚段ID
mutex | ib_mutex_t | 互斥量
space | ulint | 回滚段segment header所在页的space
page_no | ulint | 回滚段segment header所在页的偏移量
max_size | ulint | 最大undo页的数量
curr_size | ulint | 当前undo页的数量
update_undo_list | UT_LIST_BASE_NODE_T(trx_undo_t) | update undo日志链表
update_undo_cache | UT_LIST_BASE_NODE_T(trx_undo_t) | 被缓存的update undo日志链表
insert_undo_list | UT_LIST_BASE_NODE_T(trx_undo_t) | insert undo日志链表
insert_undo_cache | UT_LIST_BASE_NODE_T(trx_undo_t) | 被缓存的insert undo日志链表
last_page_no | ulint | 最近一次未被purge的undo日志header所在页的page number
last_offset | ulint | 最近一次未被purge的undo日志header在页内的偏移位置
last_trx_no | trx_id_t | 最新未被purge的事务编号

undo段中存储的是页类型为undo的页，包含undo log page header；undo log segment header仅保存在undo段的第一个undo页中。如下图所示：

![image03]({{site.baseurl}}/image/20180615/undo_pages.png)

undo段的内存数据结构为trx_undo_t:

变量 | 类型 | 说明
:----|:----|:----
id | ulint | undo段ID
type | ulint | undo日志类型，有效值TRX_UNDO_INSERT或TRX_UNDO_UPDATE
state | ulint | undo段状态
del_marks | ibool | 事务进行了delete mark操作或者更新了extern属性的列，则为TRUE
trx_id | trx_id_t | 使用当前undo段的事务ID
dict_operation | ibool | 是否为DDL操作
table_id | table_id_t | 若为DDL操作，对应表ID
rseg | trx_rseg_t* | undo段对应回滚段
space | ulint | 存放undo日志页的space id
hdr_page_no | ulint | undo log header所在表空间中的偏移量
hdr_offset | ulint | undo log header在页中的偏移量
last_page_no | ulint | undo段最新使用undo页的page no
size | ulint | 当前undo段中undo页的数量
empty | ulint | undo日志是否为空，若事务仅是SELECT操作则没有undo日志产生
top_page_no | ulint | 最后一个undo日志所在页的偏移
top_offset | ulint | 最后一个undo日志在页内的偏移量
top_undo_no | undo_no_t | 最后一个undo日志的undo no
guess_block | buf_block_t* | 最后一个undo日志在缓冲池中的页
undo_list | UT_LIST_NODE_T(trx_undo_t) | undo链表

InnoDB存储引擎允许在一个页中存放多个不同事务的undo日志。当事务提交时，首先将undo log放入到链表中，然后如果undo页使用空间小于3/4（TRX_UNDO_PAGE_REUSE_LIMIT），表示该undo页可以被重用（TRX_UNDO_STATE状态被设置为TRX_UNDO_CACHED）。由于undo页可能存放着不同事务的undo日志，因此purge操作涉及磁盘的离散读取操作。另外，重用undo页的另一个条件是当前undo段中只有一个undo页（为什么？函数trx_undo_set_state_at_finish）

## undo记录

undo日志以逻辑的方式进行存储。每个undo记录由两部分组成，undo log header和undo log record。undo log header用于保存每个事务undo日志通用信息，undo log record可以有update undo log record和insert undo log record。insert undo log record对应INSERT操作，在事务提交后就能被删除，不需要进行purge操作；update undo log record对应UPDATE/DELETE操作，事务提交时会放入回滚段的TRX_RSEG_HISTORY链表的头部，然后等待在purge线程中被清理。

每个事务发生DML操作时，由函数trx_undo_create给事务分配一个undo log header。INSERT和UPDATE操作不能存放在一个undo页中，每个类型undo记录需要分配单独的undo段。通过undo日志就可以构造之前记录版本，由函数trx_undo_prev_version_build实现。

# **Purge操作**

Purge操作主要进行2个清理操作：
- 清理已经标记为Delete Mark的记录或者其他相关辅助索引记录
- 清理undo日志，若undo页中所有的undo记录都被删除，则删除对应的undo段。需要判断当前是否有其他事务正在通过undo日志进行多版本控制，若有不能立即进行undo日志的清理。

事务提交时，undo日志有序的存放在回滚段的HISTORY链表中（即TRX_RSEG_HISTORY处）。在undo段内，undo日志是根据事务的顺序存放的。其结构大致如下图：

![image03]({{site.baseurl}}/image/20180615/trx_undo.png)

数据结构trx_purge_t：

变量 | 类型 | 说明
:----|:----|:----
trx | trx_t* | purge同样被视为一个事务，类型为TRX_PURGE
latch | rw_lock_t | 保护purge的read_view_t对象
event | os_event_t | State信号事件
running | volatile bool | 保护DROP TABLE操作
view | read_view_t* | purge操作不能删除大于该对象的undo日志
next_stored | ibool | 如果下一个undo log record已经被读取，则该变量为TRUE
rseg | trx_rseg_t* | 下一个undo log record所在的回滚段
page_no | ulint | 下一个undo log record所在的页
offset | ulint | 下一个undo log record所在的页内偏移
hdr_page_no | ulint | 下一个undo log record的log header所在的页，可能和page_no不同
hdr_offset | ulint | 下一个undo log record的log header在页内的偏移
heap | mem_heap_t* | 内存堆，用于临时内存管理
ib_bh | ib_bh_t* | 最小堆
bh_mutex | ib_mutex_t | 保护最小堆

变量running用于保护DROP TABLE操作，当进行purge undo日志时需持有该对象的x-latch；当进行DROP TABLE操作时，需持有该对象s-latch，以保证当进行purge操作时，表不会被删除。变量latch用于确保删除undo日志的正确性，因为删除时要求没有其他事务引用该undo日志。

# ** 回滚指针 **

列roll_ptr指向当前记录的undo log record，需要7个字节大小的空间：undo段id（1 byte），page_no（4 bytes）和offset（2 bytes）。

通常来说，当一个事务运行过程中遇到问题，如违反唯一约束，事务仅回滚最近一个SQL语句，这是事务状态依然是活跃的，然后用户显示的通过COMMIT或者ROLLBACK命令让事务提交或者完全回滚。但是当发生死锁或者超时的情况，InnoDB会回滚整个事务。

Rollback取得undo log record后，根据不同类型日志，进行逆操作。对于insert undo log record，逆操作DELETE是真正的删除操作，而不是DELETE MARK伪删除。因为事务没有提交，记录对其他事务是不可见的。另外，应首先删除对应的辅助索引中的记录，然后删除聚集索引中的记录。