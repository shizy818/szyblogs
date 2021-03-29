---
title:  “InnoDB 内核阅读笔记（八）- 锁”
mathjax: true
layout: post
date:   2018-05-20 08:00:12 +0800
categories: database
---

在数据库系统中，与锁互为等价概念的：
- 并发控制（concurrency control）
- 序列化（serializability）
- 隔离性（isolation）
锁是用来实现事务一致性和隔离性的一种常用技术。

# **lock文件夹**

文件名称 |  说明
:----|:----
lock0iter.cc | 队列迭代器锁
lock0lock.cc | 锁相关代码
lock0wait.cc |


# **综述**

SQL标准定义的四个隔离级别：
- READ UNCOMMITTED
- READ COMMITTED
- REPEATABLE READ
- SERIALIZABLE

UNCOMMITTED称为浏览访问，仅对于只读事务而言；READ COMMITTED称为游标稳定（cusor stability）；REPEATABLE READ是2.9999隔离，没有幻读保护；SERIALIZABLE称为3的隔离。

InnoDB存储引擎默认隔离级别是REPEATABLE READ，而且通过next-key locking算法，在事务隔离级别REPEATABLE READ下避免幻读，即已经达到了SQL标准的SERIALIZABLE。

InnoDB锁类型：
- 行级锁（Row Lock）: 共享锁（S Lock）\| 排它锁（X Lock）
- 表锁/意向锁（Intention Lock)：共享锁（S Lock）\| 排它锁（X Lock）\| 意向共享锁（IS Lock）\| 意向排它锁（IX Lock）

另外InnoDB支持3种行锁算法：
- 记录锁（Record Lock）
- 间隙锁（Gap Lock）
- Next-key Lock

# **锁数据结构**

```cpp
/** A table lock */
struct lock_table_t {
	dict_table_t*	table;		/*!< database table in dictionary
					cache */
	UT_LIST_NODE_T(lock_t)
			locks;		/*!< list of locks on the same
					table */
};

/** Record lock for a page */
struct lock_rec_t {
	ulint	space;			/*!< space id */
	ulint	page_no;		/*!< page number */
	ulint	n_bits;			/*!< number of bits in the lock
					bitmap; NOTE: the lock bitmap is
					placed immediately after the
					lock struct */
};
```

行锁由数据结构lock_rec_t表示。要知道页中某一条记录是否已经有锁，通过位图，如果值为1表示该记录已经持有锁，位图中的索引与记录heap_no对应。示意图如下。表锁分为意向锁和自增锁两大类，由数据结构lock_table_t表示。  
![image01]({{site.baseurl}}/image/20180520/lock_bitmap.png)

实际产生的锁是在事务中，对应每个行锁或者表锁，有一个锁结构lock_t。有两种途径查询锁：第一种方式通过trx_t对象的trx_locks链表进行遍历可得到某个事务所持有或者正在等待的锁信息；另一种方式中全局变量lock_sys本质是一个哈希表，根据记录所在页进行哈希查询得到lock_rec_t，再扫描lock位图判断，最终得到该行记录是否有锁。

![image02]({{site.baseurl}}/image/20180520/lock_structure.png)

```cpp
/** Lock struct; protected by lock_sys->mutex */
struct lock_t {
	trx_t*		trx;		/*!< transaction owning the
					lock */
	UT_LIST_NODE_T(lock_t)
			trx_locks;	/*!< list of the locks of the
					transaction */
	ulint		type_mode;	/*!< lock type, mode, LOCK_GAP or
					LOCK_REC_NOT_GAP,
					LOCK_INSERT_INTENTION,
					wait flag, ORed */
	hash_node_t	hash;		/*!< hash chain node for a record
					lock */
	dict_index_t*	index;		/*!< index for a record lock */
	union {
		lock_table_t	tab_lock;/*!< table lock */
		lock_rec_t	rec_lock;/*!< record lock */
	} un_member;			/*!< lock details */
};
```

对于lock_sys对象，mutex和wait_mutex分别对锁对象操作和等待线程队列进行并发保护。

在lock_t中，锁的类型（LOCK_TABLE或是LOCK_REC）和模式（LOCK_S/LOCK_IS/LOCK_X/LOCK_IX/LOCLOCK_AUTO_INC）保存在一个字段type_mode中。宏LOCK_MODE_MASK，LOCK_TYPE_MASK分别用来提取对应锁的模式和类型。另外还有标志位信息：
- LOCK_WAIT：当前请求没有被授权，处于等待模式
- LOCK_ORDINARY：Next-key锁
- LOCK_GAP
- LOCK_REC_NOT_GAP
- LOCK_INSERT_INTENTION

官方解释，LOCK_INSERT_INTENTION是特殊的GAP锁，当多事务并发插入相同的间隙空间时，只要插入记录不是相同位置（唯一键冲突），则事务间无须相互等待。为什么不直接使用GAP锁呢？当查询事务获取GAP锁或者Next-Key锁，插入事务若请求GAP锁是兼容的，将造成查询事务的幻读。设计LOCK_INSERT_INTENTION锁，当插入位置的下一条记录已经有GAP锁，则等待Gap锁释放。然而反之，LOCK_INSERT_INTENTION锁不阻止任何准备加的锁。

# **加锁场景分析**

简单的查询操作比如`SELECT * FROM tb WHERE ?`属于快照读，不需要加锁。此处分析主要针对当前读（Current Read）：
- `SELECT * FROM tb WHERE ? LOCK IN SHARE MODE`
- `SELECT * FROM tb WHERE ? FOR UPDATE`
- `INSERT INTO tb VALUES(...)`
- `UPDATE tb SET ? WHERE ?`
- `DELETE FROM tb WHERE ?`

1. 主键 + Read Committed  
	满足条件的聚集索引记录上数据锁

2. 辅助唯一索引 + Read Committed  
	满足条件的辅助索引记录上数据锁，对应聚集索引记录上数据锁

3. 辅助索引非唯一索引 + Read Committed  
	满足条件的辅助索引记录上数据锁；第一条不满足条件的记录，等值查询不上锁，范围查询上数据锁。对应聚集索引记录上数据锁

	![image03]({{site.baseurl}}/image/20180520/rc_lock.png)

4. 无索引列查询 + Read Committed  
	全表扫描，每条记录均上数据锁；不满足条件记录，在判断后放锁（semi-consistent read）

5. 主键 + Repeatable Read  
	等值查询，满足条件的聚集索引记录上数据锁；范围查询，满足条件的聚集索引记录上Next-Key锁或者GAP锁

6. 辅助唯一索引 + Repeatable Read  
	等值查询，满足条件的辅助索引记录上数据锁；范围查询，满足条件的辅助索引记录上Next-Key锁或者GAP锁。对应聚集索引记录上数据锁

7. 辅助非唯一索引 + Repeatable Read  
	满足条件的辅助索引记录上Next-key锁；第一条不满足条件的记录，等值查询上GAP锁，范围查询上Next-Key锁。对应聚集索引记录上数据锁

	![image04]({{site.baseurl}}/image/20180520/rr_lock.png)

8. 无索引列查询 + Repeatable Read  
	全表扫描上锁。如果设置了innodb_locks_unsafe_for_binlog参数，通过semi-consistent read提前放锁


# **显式锁和隐式锁**

InnoDB中存在两种属性的锁，显式锁（Explicit Lock）和隐式锁（Implicit Lock）。隐式锁总是x-lock，表示索引记录逻辑上有x-lock，但实际在内存对象中并不包含这个锁信息。其目标是减少Insert时的加锁开销，减小内存开销。后续扫描（当前读）遇到隐式锁，将其转换为显式锁。
- 聚集索引  
	用户插入一条新记录，但事务还未提交，此时该记录包含一个隐式锁。聚集索引记录有一个事务id隐藏列，若事务仍为活跃事务，则此记录有隐藏锁；反之，则不包含隐藏锁。
- 辅助索引  
	如果当前对聚集索引某记录进行了更改，而且更改的列是辅助索引列，则辅助索引上包含一个隐式锁。当辅助索引页PAGE_MAX_TRX_ID小于当前最小活跃事务，则已经提交事务修改了辅助索引记录，不含有隐藏锁。PAGE_MAX_TRX_ID大于等于当前最小活跃事务的情况：  
	![image05]({{site.baseurl}}/image/20180520/implicit_lock.png)

	* 若聚集索引记录trx_id为不活跃事务，辅助索引记录不含隐藏锁
	* 若prev_version是NULL，表示没有之前版本记录，辅助索引记录是当前事务插入的记录，则辅助索引记录含有隐藏锁
	* rec==entry，两个版本的辅助索引记录相等，但两个记录delete flag位不同，表示活跃事务删除了辅助索引记录，因此辅助索引记录rec含有隐藏锁
	* rec!=entry，两个版本的辅助索引记录不相同，且记录delete flag为0，表示活跃事务更新了辅助索引记录，因此辅助索引记录rec含有隐藏锁
	* rec==entry，但两个记录delete flag相同，既可能是当前活跃事务修改了辅助索引记录，也可能是已提交事务修改。
	* rec!=entry，且rec的delete flag为1，既可能是当前活跃事务修改了辅助索引记录，也可能是已提交事务修改。
	

# **加锁过程**

对表加锁通过函数lock_table完成，调用lock_table_create初始化表锁对象lock_t，将lock_t加入dict_table_t的链表中。

对行记录(READ/UPDATE/DELETE)通过函数lock_rec_lock完成，参数impl表示是否为隐式锁。若是隐式锁，并且当前没有该锁的冲突模式存在，则不需要产生一个lock_t的锁对象；否则通过lock_rec_create创建一个lock_t对象，并加入等待队列。InnoDB可以重用已经创建的lock_t对象，前提是同一事务锁住的同一个页面的记录，而且锁的模式相同。

函数lock_rec_lock首先尝试加快锁lock_rec_lock_fast，针对该页还没有显式锁或者hash表里仅有一个lock_t锁对象的常见情况：
```cpp
UNIV_INLINE
enum lock_rec_req_status
lock_rec_lock_fast(
/*===============*/
	ibool			impl,	/*!< in: if TRUE, no lock is set
					if no wait is necessary: we
					assume that the caller will
					set an implicit lock */
	ulint			mode,	/*!< in: lock mode: LOCK_X or
					LOCK_S possibly ORed to either
					LOCK_GAP or LOCK_REC_NOT_GAP */
	const buf_block_t*	block,	/*!< in: buffer block containing
					the record */
	ulint			heap_no,/*!< in: heap number of record */
	dict_index_t*		index,	/*!< in: index of record */
	que_thr_t*		thr)	/*!< in: query thread */
{
	lock_t*			lock;
	trx_t*			trx;
	enum lock_rec_req_status status = LOCK_REC_SUCCESS;

	ut_ad(lock_mutex_own());
	ut_ad((LOCK_MODE_MASK & mode) != LOCK_S
	      || lock_table_has(thr_get_trx(thr), index->table, LOCK_IS));
	ut_ad((LOCK_MODE_MASK & mode) != LOCK_X
	      || lock_table_has(thr_get_trx(thr), index->table, LOCK_IX));
	ut_ad((LOCK_MODE_MASK & mode) == LOCK_S
	      || (LOCK_MODE_MASK & mode) == LOCK_X);
	ut_ad(mode - (LOCK_MODE_MASK & mode) == LOCK_GAP
	      || mode - (LOCK_MODE_MASK & mode) == 0
	      || mode - (LOCK_MODE_MASK & mode) == LOCK_REC_NOT_GAP);
	ut_ad(dict_index_is_clust(index) || !dict_index_is_online_ddl(index));

	DBUG_EXECUTE_IF("innodb_report_deadlock", return(LOCK_REC_FAIL););

	// 根据space和page_no找到哈希表中第一个lock_t
	lock = lock_rec_get_first_on_page(block);

	trx = thr_get_trx(thr);

	if (lock == NULL) {
		// 如果lock不存在并且不是隐式锁，调用lock_rec_create创建
		if (!impl) {
			/* Note that we don't own the trx mutex. */
			lock = lock_rec_create(
				mode, block, heap_no, index, trx, FALSE);

		}
		status = LOCK_REC_SUCCESS_CREATED;
	} else {
		trx_mutex_enter(trx);
		// 以下情况均返回失败：
		// 哈希表中有多个该页lock_t对象的复杂情况
		// 或者锁的事务不是当前事务
		// 或者锁的类型不是mode | LOCK_REC（行锁）
		// 或者heap_no大于n_bits，意味着锁个数超出了锁位图个数
		if (lock_rec_get_next_on_page(lock)
		     || lock->trx != trx
		     || lock->type_mode != (mode | LOCK_REC)
		     || lock_rec_get_n_bits(lock) <= heap_no) {

			status = LOCK_REC_FAIL;
		} else if (!impl) {
			/* If the nth bit of the record lock is already set
			then we do not set a new lock bit, otherwise we do
			set */
			if (!lock_rec_get_nth_bit(lock, heap_no)) {
				lock_rec_set_nth_bit(lock, heap_no);
				status = LOCK_REC_SUCCESS_CREATED;
			}
		}

		trx_mutex_exit(trx);
	}

	return(status);
}
```
然后加慢锁：
```cpp
static
dberr_t
lock_rec_lock_slow(
/*===============*/
	ibool			impl,	/*!< in: if TRUE, no lock is set
					if no wait is necessary: we
					assume that the caller will
					set an implicit lock */
	ulint			mode,	/*!< in: lock mode: LOCK_X or
					LOCK_S possibly ORed to either
					LOCK_GAP or LOCK_REC_NOT_GAP */
	const buf_block_t*	block,	/*!< in: buffer block containing
					the record */
	ulint			heap_no,/*!< in: heap number of record */
	dict_index_t*		index,	/*!< in: index of record */
	que_thr_t*		thr)	/*!< in: query thread */
{
	trx_t*			trx;
	dberr_t			err = DB_SUCCESS;

	ut_ad(lock_mutex_own());
	ut_ad((LOCK_MODE_MASK & mode) != LOCK_S
	      || lock_table_has(thr_get_trx(thr), index->table, LOCK_IS));
	ut_ad((LOCK_MODE_MASK & mode) != LOCK_X
	      || lock_table_has(thr_get_trx(thr), index->table, LOCK_IX));
	ut_ad((LOCK_MODE_MASK & mode) == LOCK_S
	      || (LOCK_MODE_MASK & mode) == LOCK_X);
	ut_ad(mode - (LOCK_MODE_MASK & mode) == LOCK_GAP
	      || mode - (LOCK_MODE_MASK & mode) == 0
	      || mode - (LOCK_MODE_MASK & mode) == LOCK_REC_NOT_GAP);
	ut_ad(dict_index_is_clust(index) || !dict_index_is_online_ddl(index));

	DBUG_EXECUTE_IF("innodb_report_deadlock", return(DB_DEADLOCK););

	trx = thr_get_trx(thr);
	trx_mutex_enter(trx);

	// 如果已经存在某个锁的模式大于请求锁模式，直接返回
	if (lock_rec_has_expl(mode, block, heap_no, trx)) {

		/* The trx already has a strong enough lock on rec: do
		nothing */

	} else if (lock_rec_other_has_conflicting(
			static_cast<enum lock_mode>(mode),
			block, heap_no, trx)) {

		// 与当前记录锁不兼容，创建一个新的lock，锁模式为LOCK_WAIT | mode
		/* If another transaction has a non-gap conflicting
		request in the queue, as this transaction does not
		have a lock strong enough already granted on the
		record, we have to wait. */

		err = lock_rec_enqueue_waiting(
			mode, block, heap_no, index, thr);

	} else if (!impl) {
		/* Set the requested lock on the record, note that
		we already own the transaction mutex. */
		
		// 与当前锁记录兼容，锁模式为 LOCK_REC | mode
		lock_rec_add_to_queue(
			LOCK_REC | mode, block, heap_no, index, trx, TRUE);

		err = DB_SUCCESS_LOCKED_REC;
	}

	trx_mutex_exit(trx);

	return(err);
}
```

函数lock_rec_has_to_wait检查兼容性：
```cpp
UNIV_INLINE
ibool
lock_rec_has_to_wait(
/*=================*/
	const trx_t*	trx,	/*!< in: trx of new lock */
	ulint		type_mode,/*!< in: precise mode of the new lock
				to set: LOCK_S or LOCK_X, possibly
				ORed to LOCK_GAP or LOCK_REC_NOT_GAP,
				LOCK_INSERT_INTENTION */
	const lock_t*	lock2,	/*!< in: another record lock; NOTE that
				it is assumed that this has a lock bit
				set on the same record as in the new
				lock we are setting */
	ibool lock_is_on_supremum)  /*!< in: TRUE if we are setting the
				lock on the 'supremum' record of an
				index page: we know then that the lock
				request is really for a 'gap' type lock */
{
	ut_ad(trx && lock2);
	ut_ad(lock_get_type_low(lock2) == LOCK_REC);

	// * 锁兼容矩阵
	// *    IS IX S  X  AI
	// * IS +	 +  +  -  +
	// * IX +	 +  -  -  +
	// * S  +	 -  +  -  -
	// * X  -	 -  -  -  -
	// * AI +	 +  -  -  -
	if (trx != lock2->trx
	    && !lock_mode_compatible(static_cast<enum lock_mode>(
			             LOCK_MODE_MASK & type_mode),
				     lock_get_mode(lock2))) {

		/* We have somewhat complex rules when gap type record locks
		cause waits */

		if ((lock_is_on_supremum || (type_mode & LOCK_GAP))
		    && !(type_mode & LOCK_INSERT_INTENTION)) {
			
			// LOCK_GAP请求与其他类型锁都兼容

			/* Gap type locks without LOCK_INSERT_INTENTION flag
			do not need to wait for anything. This is because
			different users can have conflicting lock types
			on gaps. */

			return(FALSE);
		}

		if (!(type_mode & LOCK_INSERT_INTENTION)
		    && lock_rec_get_gap(lock2)) {

			// LOCK_ORDINARY/LOCK_REC_NOT_GAP请求与LOCK_GAP兼容
			/* Record lock (LOCK_ORDINARY or LOCK_REC_NOT_GAP
			does not need to wait for a gap type lock */

			return(FALSE);
		}

		if ((type_mode & LOCK_GAP)
		    && lock_rec_get_rec_not_gap(lock2)) {

			// LOCK_GAP请求与LOCK_REC_NOT_GAP兼容
			/* Lock on gap does not need to wait for
			a LOCK_REC_NOT_GAP type lock */

			return(FALSE);
		}

		if (lock_rec_get_insert_intention(lock2)) {

			// 持有LOCK_INSERT_INTENTION锁与其他类型锁请求兼容

			/* No lock request needs to wait for an insert
			intention lock to be removed. This is ok since our
			rules allow conflicting locks on gaps. This eliminates
			a spurious deadlock caused by a next-key lock waiting
			for an insert intention lock; when the insert
			intention lock was granted, the insert deadlocked on
			the waiting next-key lock.

			Also, insert intention locks do not disturb each
			other. */

			return(FALSE);
		}

		return(TRUE);
	}

	return(FALSE);
}
```

# **行锁的维护**

## 插入

例如当前有记录1, 2, 3, 4, 5, 7, 8，需要插入6这条记录，首先根据查询模式PAGE_CUR_LE定位到记录5，接着判断下一条记录7是否有区间锁或者Next-Key锁。若记录7上有锁，则不允许在这个范围进行插入操作，插入记录6的操作将被阻塞，调用函数lock_rec_enqueue_waiting等待锁释放；若记录7上没有锁，则直接插入，不产生任何锁对象。

此外，若记录7上有锁，插入操作完成后，无须事务提交，调用函数lock_update_insert更新锁定范围，原锁定范围从（5，7]更新为(5, 6)和（6，7]。若插入的表上有辅助索引，还需要对辅助索引记录进行锁的判断，判断可以进行插入后，更新辅助索引页的PAGE_MAX_TRX_ID值。

函数lock_rec_insert_check_and_lock检查插入锁冲突，并对插入位置的下一条记录上锁 `LOCK_X | LOCK_GAP | LOCK_INSERT_INTENTION`。参数inherit用来判断是否在插入完成后调用函数lock_update_insert对已锁定范围更新。

插入记录默认是隐式锁，当有其他事务操作同一聚集索引记录，调用函数lock_rec_convert_impl_to_expl将隐式锁转换为显式锁，模式为`LOCK_REC | LOCK_X | LOCK_REC_NOT_GAP`。若并行事务准备插入相同主键记录，函数row_ins_duplicate_error_in_clust检查重复键，并请求共享锁`LOCK_REC | LOCK_S | lock_type(TRX_ISO_READ_COMMITTED ? LOCK_REC_NOT_GAP : LOCK_ORDINARY)`。由此从代码可以解释前文[Innodb Gap Lock](/szyblogs/database/2018/04/10/Innodb+Gap+Lock/)中的死锁情况。

## 更新

记录的更新或删除操作，首先尝试对记录加X隐式锁；若更新记录上存在其他锁时，事务被阻塞，需要等待记录上的锁被释放。

函数lock_clust_rec_modify_check_and_lock和lock_sec_rec_modify_check_and_lock分别对聚集索引和辅助索引进行加锁。两者都调用lock_rec_lock函数，不同在于：
- 聚集索引记录加锁前需要将记录上的implicit lock转化为explicit lock
- 辅助索引记录加锁成功后，需要更新辅助索引页page header中的PAGE_MAX_TRX_ID

在更新记录过程中，如果不能原地更新，可以通过调用lock_rec_store_on_page_infimum将锁信息临时保存在伪记录infimum上。

## Purge

对于记录的删除，首先是将delete flag标志位设置为1，然后通过后台的purge线程调用函数page_cur_delete_rec将记录真正的删除。当purge真正删除记录操作完成后，下一条记录需要通过函数lock_update_delete继承删除记录的锁定范围。

## 页的分裂与合并

往右分裂时，函数lock_update_split_right对锁进行维护步骤如下：
1. 确定分裂点记录split_rec
2. 调用函数lock_rec_move将记录split_rec到记录supremum之间的所有锁移动到新页中，修改lock位图中的值
3. 调用函数lock_rec_move将原来页记录supremum持有的锁移动到新页supremum记录
4. 调用函数lock_rec_inherit_to_gap将新页第一条记录的锁继承给原页的记录supremem（类型LOCK_GAP）

向左分裂类似，通过函数lock_update_split_left实现。页合并的实现包括lock_update_merge_left和lock_update_merge_right。

参考：  
[何登成：InnoDB事务/锁/多版本分析](https://wenku.baidu.com/view/3fd448e9f8c75fbfc77db2ca.html)  
[MySQL加锁处理分析](http://blog.jobbole.com/99413/)  
[一个最不可思议的MySQL死锁分析](http://blog.jobbole.com/99401/)