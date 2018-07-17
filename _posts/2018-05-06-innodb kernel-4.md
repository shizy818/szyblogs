---
title:  “InnoDB 内核阅读笔记（四）- mini事务”
date:   2018-05-06 08:00:12 +0800
categories: database
---

mini-transaction用来保证并发事务操作下数据页的一致性。mini-transaction的形式如下：
```
mini_transaction() {
	x-lock page
	transform the page
	generate undo/redo log record
	unlock x-lock
}
```

**注**： 若操作页是B+树索引的非叶子节点页，那么由于非叶子节点通过相应B+树索引的latch保护，因此操作这些页时不需要持有页的latch。


# **mtr文件夹**

文件名称 |  说明
:----|:----
mtr0log.cc | mini-transaction日志操作
mtr0mtr.cc | mini-transaction基本操作

# **数据结构**

mtr_t结构：
```cpp
/* Mini-transaction handle and buffer */
struct mtr_t{
#ifdef UNIV_DEBUG
	ulint		state;	/*!< MTR_ACTIVE, MTR_COMMITTING, MTR_COMMITTED */
#endif
	dyn_array_t	memo;	/*!< memo stack for locks etc. */
	dyn_array_t	log;	/*!< mini-transaction log */
	unsigned	inside_ibuf:1;
				/*!< TRUE if inside ibuf changes */
	unsigned	modifications:1;
				/*!< TRUE if the mini-transaction
				modified buffer pool pages */
	unsigned	made_dirty:1;
				/*!< TRUE if mtr has made at least
				one buffer pool page dirty */
	ulint		n_log_recs;
				/* count of how many page initial log records
				have been written to the mtr log */
	ulint		n_freed_pages;
				/* number of pages that have been freed in
				this mini-transaction */
	ulint		log_mode; /* specifies which operations should be
				logged; default value MTR_LOG_ALL */
	lsn_t		start_lsn;/* start lsn of the possible log entry for
				this mtr */
	lsn_t		end_lsn;/* end lsn of the possible log entry for
				this mtr */
#ifdef UNIV_DEBUG
	ulint		magic_n;
#endif /* UNIV_DEBUG */
};
```

变量 | 类型 |  说明
:----|:----|:----
state | ulint | 有效状态：MTR_ACTIVE / MTR_COMMITTING / MTR_COMMITTED
memo | dyn_array_t | 持有latch信息，保存数据结构为mtr_memo_slot_t
log | dyn_array_t | 产生的mini-transaction日志
modifications | unsigned | 事务是否更改页
n_log_recs | ulint | 有多少页的日志被写入变量log中
log_mode | ulint | 有效值：MTR_LOG_ALL / MTR_LOG_SHORT_INSERTS/ MTR_LOG_NONE / MTR_LOG_NO_REDO
start_lsn | lsn_t | mini-transaction开始LSN
end_lsn | lsn_t | mini-transaction结束LSN

InnoDB存储引擎的重做日志是物理逻辑的，physical-to-a-page & logical-within-a-page。比如MLOG_REC_INSERT类型日志，并不直接保存插入的记录，而是插入记录的前一个记录位置，以及插入记录与前一条记录的差异信息。

mini-transaction的使用方式：
```cpp
mtr_t mtr;
mtr_start(&mtr)
......
mtr_commit(&mtr)
```

mtr_commit里调用mtr_log_reserve_and_write函数将mtr中的日志写入
重做日志缓冲里，这时需要持有log_sys->mutex；将脏页添加到flush list后，调用log_release释放log_sys->mutext；调用函数mtr_memo_pop_all释放所有latch；

互斥量log_sys->mutex是一个热点，mini-transaction写入重做日志缓冲需要持有该互斥量，从重做日志缓冲写入到文件也需要持有该互斥量。从重做日志缓冲写入到文件是缓存写的方式，因此在fsync操作前就释放了log_sys->mutex,c从而实现事务的组提交。当一个事务正在提交进行fsync时，其他事务可以获得log_sys->mutex对象，并将事务日志写入重做日志缓冲，待下一次事务提交fsync时，可以将多个事务的重做日志一次性写入到重做日志文件中。