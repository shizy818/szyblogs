---
title:  “InnoDB 内核阅读笔记（三）- 重做日志”
mathjax: true
layout: post
date:   2018-05-01 08:00:12 +0800
categories: database
---

重做日志（redo log)用来实现事务的持久性，即当事务提交时，必须先确保将事务所有日志写入重做日志文件，
称之为force log at commit。另外，在脏页刷新到磁盘前，所有内存中小于该页LSN的日志都必须刷新到磁盘，即write ahead log(WAL)。

为了确保每次日志都写入重做日志文件，在每次将重做日志缓冲写入到重做日志文件后，InnoDB存储引擎需要调用一次fsync操作。
因为重做日志文件打开并没有使用O_DIRECT选项，所以重组日志先写入到文件系统缓存。为了确保重做日志写入磁盘，必须进行
一次fsync操作。fsync的效率取决于磁盘性能，因此磁盘的性能决定了事务提交的性能。为了提高数据库提交时的性能，
数据库允许将一组事务进行提交，称之为组提交(group commit)。


InnoDB引擎允许用户通过`innodb_flush_log_at_trx_commit`参数来控制重组日志的刷盘策略。1为默认值，
表示事务提交时必须将该事务所有日志写入磁盘；0表示事务提交时并不强制所有日志写入磁盘，这个操作在master thread中进行，
由于master thread每1秒进行一次fsync操作，因此可能会丢失1秒内的重做日志；2表示重做日志仅写入到文件系统的缓存中，
如果MySQL宕机但操作系统不宕机，并不会导致事务数据的丢失。

MySQL数据库还有一种称为二进制日志（binary log），用于point-in-time(PIT)恢复以及主从复制(replication)。
然而二进制日志是MySQL上层生成的，不仅仅针对InnoDB引擎，且其记录对应SQL语句（二进制名不符实）。

事务的日志在提交时确定写到外存，但是缓冲池（buffer pool）中的页并没有刷新到磁盘，检查点所做的操作就是将缓冲池
中的页刷新到磁盘，这样可以缩短宕机时数据库恢复所需的时间。InnoDB存储引擎存在两种检查点：
- sharp checkpoint： 将脏页全部刷新到磁盘
- fuzzy checkpoint： 根据脏页第一次被修改时的LSN排序，将最老的脏页先刷回到磁盘

# **log文件夹**

文件名称 |  说明
:----|:----
log0log.cc | 重做日志相关操作
log0recv.cc | 重做日志恢复

# **重做日志物理架构**

InnoDB存储引擎中关于重做日志几个概率：
- 重做日志缓存 （redo log buffer)
- 重做日志组 (redo log group)
- 重做日志文件 (redo log file)
- 重做日志块（redo log block）
- 归档日志文件 (archive log file)

每个重做日志文件组包含多个重做日志文件，每个重做日志组的存储内容是完全相同的，是镜像的关系，
目的是提供数据库的可用性。但似乎MySQL并没有实现此功能。   
![image01]({{site.baseurl}}/image/20180501/redo_log_group.png)

重做日志块以512字节进行存储，这个大小跟磁盘扇区大小一样，因此重做日志的写入可以保证原子性，
不需要doublewrite技术。重做日志缓存由重做日志块组成，在内部就好似一个数组。

重做日志缓存刷盘的时间点：
- 事务提交
- 检查点
- 日志缓存中已使用空间超过某个阈值

每个重做日志组的第一个重做日志文件中，记录了额外的2KB信息，包含了log file header和检查点，
用于InnoDB存储引擎的宕机恢复操作。所以除了顺序追加写入重做日志块，还需要更新前2KB部分的信息。

存储内容 |  大小（字节）
:----|:----
log file header | 512
checkpoint 1 | 512
空 | 512
checkpoint 2 | 512

## log_t数据结构

```cpp
/** Redo log buffer */
struct log_t{
	byte		pad[64];	/*!< padding to prevent other memory
					update hotspots from residing on the
					same memory cache line */
	lsn_t		lsn;		/*!< log sequence number */
	ulint		buf_free;	/*!< first free offset within the log
					buffer */
#ifndef UNIV_HOTBACKUP
	ib_mutex_t		mutex;		/*!< mutex protecting the log */

	ib_mutex_t		log_flush_order_mutex;/*!< mutex to serialize access to
					the flush list when we are putting
					dirty blocks in the list. The idea
					behind this mutex is to be able
					to release log_sys->mutex during
					mtr_commit and still ensure that
					insertions in the flush_list happen
					in the LSN order. */
#endif /* !UNIV_HOTBACKUP */
	byte*		buf_ptr;	/* unaligned log buffer */
	byte*		buf;		/*!< log buffer */
	ulint		buf_size;	/*!< log buffer size in bytes */
	ulint		max_buf_free;	/*!< recommended maximum value of
					buf_free, after which the buffer is
					flushed */
 #ifdef UNIV_LOG_DEBUG
	ulint		old_buf_free;	/*!< value of buf free when log was
					last time opened; only in the debug
					version */
	ib_uint64_t	old_lsn;	/*!< value of lsn when log was
					last time opened; only in the
					debug version */
#endif /* UNIV_LOG_DEBUG */
	ibool		check_flush_or_checkpoint;
					/*!< this is set to TRUE when there may
					be need to flush the log buffer, or
					preflush buffer pool pages, or make
					a checkpoint; this MUST be TRUE when
					lsn - last_checkpoint_lsn >
					max_checkpoint_age; this flag is
					peeked at by log_free_check(), which
					does not reserve the log mutex */
	UT_LIST_BASE_NODE_T(log_group_t)
			log_groups;	/*!< log groups */

#ifndef UNIV_HOTBACKUP
	/** The fields involved in the log buffer flush @{ */

	ulint		buf_next_to_write;/*!< first offset in the log buffer
					where the byte content may not exist
					written to file, e.g., the start
					offset of a log record catenated
					later; this is advanced when a flush
					operation is completed to all the log
					groups */
......
}
```

变量 | 类型 |  说明
:----|:----|:----
pad[64] | byte | 填充使得log_t对象可以放在同一cache line中，减少竞争（？？不懂？？）
lsn | lsn_t | 重做日志缓冲的LSN
buf_free | ulint | 当前已经写入到重做日志缓冲的位置
mutex | ib_mutex_t | innodb互斥
log_flush_order_mutex | ib_mutex_t | 同步flush list; 当mini-transaction释放log_sys->mutex时候，保证flush list插入LSN仍是顺序的
buf | byte* | 重做日志缓冲空间
buf_size | ulint | 重做日志缓冲空间大小
max_buf_free | ulint | 最大可用空间
check_flush_or_checkpoint | ibool | 该变量设置为TRUE，若需要刷新重做日志到文件或者需要做检查点
log_groups | UT_LIST_BASE_NODE_T(log_group_t) | 重做日志组链表
buf_next_to_write | ulint | 从这个位置开始的重做日志缓冲还未被写入到文件
written_to_some_lsn | lsn_t | 重做日志缓冲至少已经被写入一个重做日志文件组的LSN
written_to_all_lsn | lsn_t | 重做日志缓冲已经被写入到所有重做日志文件组的LSN
write_lsn | lsn_t | 已经写入到重做日志文件的LSN
write_end_offset | ulint | 已经写入到重做日志文件的偏移量，当前write结束时，该值复制到buf_next_to_write
current_flush_lsn | lsn_t | 当前正在执行刷新操作的LSN
flushed_to_disk_lsn | lsn_t | 已经刷新到磁盘的LSN
n_pending_writes | ulint | 异步写入重做日志文件操作数量
no_flush_event | os_event_t | 等待所有异步I/O写入到重做日志组的事件
one_flushed | ibool | 写入重做日志组时首先设置为FALSE，若有一个重做日志文件组完成写入，则设置为TRUE
one_flushed_event | os_event_t | 等待一个异步I/O写入到重做日志文件组的事件
n_log_ios | ulint | 已经发生的重做日志I/O操作次数
n_log_ios_old | ulint | 上次写入重做日志文件时，已经发生的重做日志I/O操作次数
max_modified_age_async | ulint | 异步刷新脏页的距离
max_modified_age_sync | ulint | 同步刷新脏页的距离
max_checkpoint_age_async | ulint | 异步写入检查点距离
max_checkpoint_age | ulint | 同步写入检查点距离
next_checkpoint_no | ib_uint64_t | 下一次检查点编号
last_checkpoint_lsn | lsn_t | 上一次检查点时的LSN
next_checkpoint_lsn | lsn_t | 下一次检查点时的LSN
n_pending_checkpoint_writes | ulint | 正在进行写入检查点操作的异步I/O数量
checkpoint_lock | rw_lock_t | latch，实现同步或异步写入操作
checkpoint_buf | byte* | 检查点重做日志块的空间

buf_free表示当前重做日志缓冲写入的开始位置; 初始化时，max_buf_free设置为差不多1/2大小的重做日志缓冲; 
buf_next_to_write表示写入到重做日志文件的位置。如果buf_free大于max_buf_free，会进行一次写入重做日志文件
的操作(log_check_margins)；然后日志缓冲里write_end_offset到buf_free之间的日志被移到缓冲区
开始的位置（log_sys_check_flush_completion）。  
![image02]({{site.baseurl}}/image/20180501/redo_buf.png)

函数log_write_low用于将每个页中的日志写入到重做日志缓冲；函数log_write_up_to将重做日志缓冲写到重做日志文件中，
参数flush_to_disk控制是否刷新到磁盘；函数log_archive_do进行重做日志的归档操作；函数log_checkpoint将
checkpoint的值写入到重做日志的头部，这里仅将检查点LSN更新到文件，并不进行脏页的刷新。

系统中有一个全局log_t对象log_sys，重做日志操作的并发控制由log_sys->mutex进行保护，因此是一个资源竞争热点。
所有重做日志的I/O操作都是异步的，这样可以提前释放log_sys->mutex。另外，写入都是先写到操作系统缓存，
然后进行fsync操作。进行fsync时，释放log_sys->mutex所持有的保护。

函数log_write_up_to中，当重做日志缓冲刷新到重做日志文件时，InnoDB存储引擎会对最后一个重做日志块进行复制。
这样，当重做日志在写入到磁盘时，因为fsync释放了log_sys->mutex，所以允许之后事务的重做日志继续写入到上次的重做日志缓冲处。

```cpp
	/* Copy the last, incompletely written, log block a log block length
	up, so that when the flush operation writes from the log buffer, the
	segment to write will not be changed by writers to the log */
	// 复制最后一个重做日志块
	ut_memcpy(log_sys->buf + area_end,
		  log_sys->buf + area_end - OS_FILE_LOG_BLOCK_SIZE,
		  OS_FILE_LOG_BLOCK_SIZE);

	log_sys->buf_free += OS_FILE_LOG_BLOCK_SIZE;
	log_sys->write_end_offset = log_sys->buf_free;

	group = UT_LIST_GET_FIRST(log_sys->log_groups);

	......

	if (srv_unix_file_flush_method == SRV_UNIX_O_DSYNC) {
		/* O_DSYNC means the OS did not buffer the log file at all:
		so we have also flushed to disk what we have written */

		log_sys->flushed_to_disk_lsn = log_sys->write_lsn;

	} else if (flush_to_disk) {

		group = UT_LIST_GET_FIRST(log_sys->log_groups);
		// 调用async刷新到磁盘
		fil_flush(group->space_id);
		log_sys->flushed_to_disk_lsn = log_sys->write_lsn;
	}
```

# **恢复**

recv_sys_t结构：
```cpp
/** Recovery system data structure */
struct recv_sys_t{
#ifndef UNIV_HOTBACKUP
	ib_mutex_t		mutex;	/*!< mutex protecting the fields apply_log_recs,
				n_addrs, and the state field in each recv_addr
				struct */
	ib_mutex_t		writer_mutex;/*!< mutex coordinating
				flushing between recv_writer_thread and
				the recovery thread. */
#endif /* !UNIV_HOTBACKUP */
	ibool		apply_log_recs;
				/*!< this is TRUE when log rec application to
				pages is allowed; this flag tells the
				i/o-handler if it should do log record
				application */
	ibool		apply_batch_on;
				/*!< this is TRUE when a log rec application
				batch is running */
	lsn_t		lsn;	/*!< log sequence number */
	ulint		last_log_buf_size;
				/*!< size of the log buffer when the database
				last time wrote to the log */
	byte*		last_block;
				/*!< possible incomplete last recovered log
				block */
	byte*		last_block_buf_start;
				/*!< the nonaligned start address of the
				preceding buffer */
	byte*		buf;	/*!< buffer for parsing log records */
	ulint		len;	/*!< amount of data in buf */
	lsn_t		parse_start_lsn;
				/*!< this is the lsn from which we were able to
				start parsing log records and adding them to
				the hash table; zero if a suitable
				start point not found yet */
	lsn_t		scanned_lsn;
				/*!< the log data has been scanned up to this
				lsn */
	ulint		scanned_checkpoint_no;
				/*!< the log data has been scanned up to this
				checkpoint number (lowest 4 bytes) */
	ulint		recovered_offset;
				/*!< start offset of non-parsed log records in
				buf */
	lsn_t		recovered_lsn;
				/*!< the log records have been parsed up to
				this lsn */
	lsn_t		limit_lsn;/*!< recovery should be made at most
				up to this lsn */
	ibool		found_corrupt_log;
				/*!< this is set to TRUE if we during log
				scan find a corrupt log block, or a corrupt
				log record, or there is a log parsing
				buffer overflow */
#ifdef UNIV_LOG_ARCHIVE
	log_group_t*	archive_group;
				/*!< in archive recovery: the log group whose
				archive is read */
#endif /* !UNIV_LOG_ARCHIVE */
	mem_heap_t*	heap;	/*!< memory heap of log records and file
				addresses*/
	hash_table_t*	addr_hash;/*!< hash table of file addresses of pages */
	ulint		n_addrs;/*!< number of not processed hashed file
				addresses in the hash table */

	recv_dblwr_t	dblwr;
};
```

addr_hash是一个哈希表对象，每个桶存放recv_addr_t数据结构，其根据（space，page_no)进行哈希放入同一个桶中，
并用链表进行串联。recv_addr_t存放对应（space, page_no)页中的重做日志recv_t；rect_t结构中记录重做日志类型，
长度，开始LSN，结束LSN，以及重做日志的body。  
![image03]({{site.baseurl}}/image/20180501/redo_recv.png)

InnoDB在表空间的首页，定义了FIL_PAGE_FILE_FLUSH_LSN，记录了数据库关闭时最后刷新页的LSN。若数据库正常关闭，
FIL_PAGE_FILE_FLUSH_LSN和重做日志中的检查点相等；若异常情况发生，两值不等，则需要进行恢复操作。

调用过程：
>	recv_recovery_from_checkpoint_start_func 
>		|
>	recv_find_max_checkpoint
> 		|
> 	log_group_read_checkpoint_info
> 		|
> 	recv_group_scan_log_recs
>		|
>	recv_group_read_log_seg
> 		|
>	recv_scan_log_recs
>		|
>	recv_sys_add_to_parsing_buf
>		|
>	recv_parse_log_recs
>		|
>	recv_parse_log_rec
>		|
>	recv_parse_or_apply_log_rec_body
>		|
>	recv_add_to_hash_table
>		|
>	recv_sys_justify_left_parsing_buf
