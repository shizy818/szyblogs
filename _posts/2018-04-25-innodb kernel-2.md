---
title:  “InnoDB 内核阅读笔记（二）- 同步机制”
date:   2018-04-25 08:00:12 +0800
categories: database
---

InnoDB存储引擎没有直接使用操作系统自带的mutex和rw-lock，而是进行了封装，并通过自旋（spin）和等待队列（wait array）的设计来提高性能。


# **sync & os 文件夹**

文件名称 |  说明
:----|:----
os0sync.cc | 操作系统同步封装
sync0arr.cc | 等待队列实现
sync0rw.cc | 读写锁实现
sync0sync.cc | 互斥量实现

# **操作系统同步机制封装**

os_fast_mutex_t封装pthread_mutex_t/CRITICAL_SECTION; os_event_t包含pthread_cond_t/CONDITION_VARIABLE,
用来实现线程同步。os_mutex_t是操作系统mutex的最终封装。

os0sync.h还定义了os_compare_and_swap和os_atomic_test_and_set等原子操作。

os_fast_mutex_t结构:
```cpp
#ifdef __WIN__
/** Native mutex */
typedef CRITICAL_SECTION	fast_mutex_t;
/** Native mutex */
typedef pthread_mutex_t		fast_mutex_t;
#endif

/** Structure that includes Performance Schema Probe pfs_psi
in the os_fast_mutex structure if UNIV_PFS_MUTEX is defined */
struct os_fast_mutex_t {
	fast_mutex_t		mutex;	/*!< os_fast_mutex */
#ifdef UNIV_PFS_MUTEX
	struct PSI_mutex*	pfs_psi;/*!< The performance schema
					instrumentation hook */
#endif
};
```

os_event_t结构:
```cpp
#ifdef __WIN__
/** Native condition variable. */
typedef CONDITION_VARIABLE	os_cond_t;
#else
/** Native condition variable */
typedef pthread_cond_t		os_cond_t;
#endif

/** Operating system event handle */
typedef struct os_event*	os_event_t;

/** An asynchronous signal sent between threads */
struct os_event {
#ifdef __WIN__
	HANDLE		handle;		/*!< kernel event object, slow,
					used on older Windows */
#endif
	os_fast_mutex_t	os_mutex;	/*!< this mutex protects the next
					fields */
	ibool		is_set;		/*!< this is TRUE when the event is
					in the signaled state, i.e., a thread
					does not stop if it tries to wait for
					this event */
	ib_int64_t	signal_count;	/*!< this is incremented each time
					the event becomes signaled */
	os_cond_t	cond_var;	/*!< condition variable is used in
					waiting for the event */
	UT_LIST_NODE_T(os_event_t) os_event_list;
					/*!< list of all created events */
};
```

os_mutex_t结构:
```cpp
/** Operating system mutex handle */
typedef struct os_mutex_t*	os_ib_mutex_t;

/* Type definition for an operating system mutex struct */
struct os_mutex_t{
	os_event_t	event;	/*!< Used by sync0arr.cc for queing threads */
	void*		handle;	/*!< OS handle to mutex */
	ulint		count;	/*!< we use this counter to check
				that the same thread does not
				recursively lock the mutex: we
				do not assume that the OS mutex
				supports recursive locking, though
				NT seems to do that */
	UT_LIST_NODE_T(os_mutex_t) os_mutex_list;
				/* list of all 'slow' OS mutexes created */
};
```

# **InnoDB同步机制**

## InnoDB mutex

InnoDB尽量不用操作系统的互斥机制，因为效率不高；所以自己实现了两种同步机制，mutex和rw-lock。

ib_mutex_t结构：
```cpp
/** InnoDB mutex */
struct ib_mutex_t {
	os_event_t	event;	/*!< Used by sync0arr.cc for the wait queue */
	volatile lock_word_t	lock_word;	/*!< lock_word is the target
				of the atomic test-and-set instruction when
				atomic operations are enabled. */

#if !defined(HAVE_ATOMIC_BUILTINS)
	os_fast_mutex_t
		os_fast_mutex;	/*!< We use this OS mutex in place of lock_word
				when atomic operations are not enabled */
#endif
	ulint	waiters;	/*!< This ulint is set to 1 if there are (or
				may be) threads waiting in the global wait
				array for this mutex to be released.
				Otherwise, this is 0. */
	UT_LIST_NODE_T(ib_mutex_t)	list; /*!< All allocated mutexes are put into
				a list.	Pointers to the next and prev. */
#ifdef UNIV_SYNC_DEBUG
	const char*	file_name;	/*!< File where the mutex was locked */
	ulint	line;		/*!< Line where the mutex was locked */
	ulint	level;		/*!< Level in the global latching order */
#endif /* UNIV_SYNC_DEBUG */
	const char*	cfile_name;/*!< File name where mutex created */
	ulint		cline;	/*!< Line where created */
	ulong		count_os_wait;	/*!< count of os_wait */
#ifdef UNIV_DEBUG

/** Value of mutex_t::magic_n */
# define MUTEX_MAGIC_N	979585UL

	os_thread_id_t thread_id; /*!< The thread id of the thread
				which locked the mutex. */
	ulint		magic_n;	/*!< MUTEX_MAGIC_N */
	const char*	cmutex_name;	/*!< mutex name */
	ulint		ib_mutex_type;	/*!< 0=usual mutex, 1=rw_lock mutex */
#endif /* UNIV_DEBUG */
#ifdef UNIV_PFS_MUTEX
	struct PSI_mutex* pfs_psi;	/*!< The performance schema
					instrumentation hook */
#endif
};
```

变量 | 类型 |  说明
:----|:----|:----
event | os_event_t | 用于等待队列
lock_word | volatile lock_word_t | 原子test-and-set（TAS）操作
os_fast_mutex | os_fast_mutex_t | 不支持TAS环境，使用操作系统互斥量保护lock_word
waiter | ulint |  是否有其他线程等待获得该mutex
list | UT_LIST_NODE_T(ib_mutex_t) | innodb mutex链表
thread_id | os_thread_id | 当前持有该mutex的线程ID

InnoDB mutex通过test-and-set设置lock_word变量：
- test-and-set如果返回0，表示获得了锁；如果返回1，进行自旋操作
- 当自旋一段时间后仍不能获得mutex，则将mutex放入wait array的槽，等待被唤醒

InnoDB实现自旋的目的在于：
- 自旋判断lock_word的值是访问CPU L1 cache或L2 cache得到的，从而可以减少对于内存的访问
- 第一次失败后不直接将线程放入等待队列，因为唤醒线程需要上下文切换。自旋可以减少上下文切换

```cpp
UNIV_INLINE
void
mutex_enter_func(
/*=============*/
	ib_mutex_t*	mutex,		/*!< in: pointer to mutex */
	const char*	file_name,	/*!< in: file name where locked */
	ulint		line)		/*!< in: line where locked */
{
	ut_ad(mutex_validate(mutex));
	ut_ad(!mutex_own(mutex));

	/* Note that we do not peek at the value of lock_word before trying
	the atomic test_and_set; we could peek, and possibly save time. */

	if (!ib_mutex_test_and_set(mutex)) {
		ut_d(mutex->thread_id = os_thread_get_curr_id());
#ifdef UNIV_SYNC_DEBUG
		mutex_set_debug_info(mutex, file_name, line);
#endif
		return;	/* Succeeded! */
	}

	mutex_spin_wait(mutex, file_name, line);
}

/******************************************************************//**
Reserves a mutex for the current thread. If the mutex is reserved, the
function spins a preset time (controlled by SYNC_SPIN_ROUNDS), waiting
for the mutex before suspending the thread. */
UNIV_INTERN
void
mutex_spin_wait(
/*============*/
	ib_mutex_t*	mutex,		/*!< in: pointer to mutex */
	const char*	file_name,	/*!< in: file name where mutex
					requested */
	ulint		line)		/*!< in: line where requested */
{
	ulint		i;		/* spin round count */
	ulint		index;		/* index of the reserved wait cell */
	sync_array_t*	sync_arr;
	size_t		counter_index;

	counter_index = (size_t) os_thread_get_curr_id();

	ut_ad(mutex);

	/* This update is not thread safe, but we don't mind if the count
	isn't exact. Moved out of ifdef that follows because we are willing
	to sacrifice the cost of counting this as the data is valuable.
	Count the number of calls to mutex_spin_wait. */
	mutex_spin_wait_count.add(counter_index, 1);

mutex_loop:

	i = 0;

	/* Spin waiting for the lock word to become zero. Note that we do
	not have to assume that the read access to the lock word is atomic,
	as the actual locking is always committed with atomic test-and-set.
	In reality, however, all processors probably have an atomic read of
	a memory word. */

spin_loop:
	os_rmb;
	while (mutex_get_lock_word(mutex) != 0 && i < SYNC_SPIN_ROUNDS) {
		if (srv_spin_wait_delay) {
			ut_delay(ut_rnd_interval(0, srv_spin_wait_delay));
		}

		i++;
	}

	if (i >= SYNC_SPIN_ROUNDS) {
		os_thread_yield();
	}

	mutex_spin_round_count.add(counter_index, i);

	if (ib_mutex_test_and_set(mutex) == 0) {
		/* Succeeded! */

		ut_d(mutex->thread_id = os_thread_get_curr_id());
#ifdef UNIV_SYNC_DEBUG
		mutex_set_debug_info(mutex, file_name, line);
#endif
		return;
	}

	/* We may end up with a situation where lock_word is 0 but the OS
	fast mutex is still reserved. On FreeBSD the OS does not seem to
	schedule a thread which is constantly calling pthread_mutex_trylock
	(in ib_mutex_test_and_set implementation). Then we could end up
	spinning here indefinitely. The following 'i++' stops this infinite
	spin. */

	i++;

	if (i < SYNC_SPIN_ROUNDS) {
		goto spin_loop;
	}

	sync_arr = sync_array_get_and_reserve_cell(mutex, SYNC_MUTEX,
						   file_name, line, &index);

	/* The memory order of the array reservation and the change in the
	waiters field is important: when we suspend a thread, we first
	reserve the cell and then set waiters field to 1. When threads are
	released in mutex_exit, the waiters field is first set to zero and
	then the event is set to the signaled state. */

	mutex_set_waiters(mutex, 1);

	/* Try to reserve still a few times */
	for (i = 0; i < 4; i++) {
		if (ib_mutex_test_and_set(mutex) == 0) {
			/* Succeeded! Free the reserved wait cell */

			sync_array_free_cell(sync_arr, index);

			ut_d(mutex->thread_id = os_thread_get_curr_id());
#ifdef UNIV_SYNC_DEBUG
			mutex_set_debug_info(mutex, file_name, line);
#endif

			return;

			/* Note that in this case we leave the waiters field
			set to 1. We cannot reset it to zero, as we do not
			know if there are other waiters. */
		}
	}

	/* Now we know that there has been some thread holding the mutex
	after the change in the wait array and the waiters field was made.
	Now there is no risk of infinite wait on the event. */

	mutex_os_wait_count.add(counter_index, 1);

	mutex->count_os_wait++;

	sync_array_wait_event(sync_arr, index);

	goto mutex_loop;
}
```

在通过mutex_set_waiters设置waiters为1后，需要再次进行test-and-set的判断，看看是否可以得到mutex。否则如下的情况，会导致线程无限等待。

时间 | 等待mutex线程 | 持有mutex线程 
:----|:----|:----
1 | mutex_enter_func |
2 | ...... |
3 | sync_array_get_and_reserve_cell | mutex_exit_func
 | # 此时mutex -> waiters = 0
4 | | mutex_reset_lock_word(mutex);
  | | if (mutex_get_waiters(mutex) != 0) { wait up waiting thread }
5 | mutex_set_waiters(mutex, 1); |
6 | ...... |
7 | sync_array_wait_event(sync_arr, index); |

## rw-lock

读写锁允许并发读，但是同一时刻只允许一个更新操作。其实此处的读写锁，更应该称之为闩锁（Latch）。因为数据库系统中，锁通常是一个更高层次的概念，比如表锁，行锁等等，持有时间可以贯穿整个事务；读写锁的作用是临界资源的共享和互斥，一般很快就会释放。

```cpp
/** The structure used in the spin lock implementation of a read-write
lock. Several threads may have a shared lock simultaneously in this
lock, but only one writer may have an exclusive lock, in which case no
shared locks are allowed. To prevent starving of a writer blocked by
readers, a writer may queue for x-lock by decrementing lock_word: no
new readers will be let in while the thread waits for readers to
exit. */
struct rw_lock_t {
	volatile lint	lock_word;
				/*!< Holds the state of the lock. */
	volatile ulint	waiters;/*!< 1: there are waiters */
	volatile ibool	recursive;/*!< Default value FALSE which means the lock
				is non-recursive. The value is typically set
				to TRUE making normal rw_locks recursive. In
				case of asynchronous IO, when a non-zero
				value of 'pass' is passed then we keep the
				lock non-recursive.
				This flag also tells us about the state of
				writer_thread field. If this flag is set
				then writer_thread MUST contain the thread
				id of the current x-holder or wait-x thread.
				This flag must be reset in x_unlock
				functions before incrementing the lock_word */
	volatile os_thread_id_t	writer_thread;
				/*!< Thread id of writer thread. Is only
				guaranteed to have sane and non-stale
				value iff recursive flag is set. */
	os_event_t	event;	/*!< Used by sync0arr.cc for thread queueing */
	os_event_t	wait_ex_event;
				/*!< Event for next-writer to wait on. A thread
				must decrement lock_word before waiting. */
#ifndef INNODB_RW_LOCKS_USE_ATOMICS
	ib_mutex_t	mutex;		/*!< The mutex protecting rw_lock_t */
#endif /* INNODB_RW_LOCKS_USE_ATOMICS */

	UT_LIST_NODE_T(rw_lock_t) list;
				/*!< All allocated rw locks are put into a
				list */
#ifdef UNIV_SYNC_DEBUG
	UT_LIST_BASE_NODE_T(rw_lock_debug_t) debug_list;
				/*!< In the debug version: pointer to the debug
				info list of the lock */
	ulint	level;		/*!< Level in the global latching order. */
#endif /* UNIV_SYNC_DEBUG */
#ifdef UNIV_PFS_RWLOCK
	struct PSI_rwlock *pfs_psi;/*!< The instrumentation hook */
#endif
	ulint count_os_wait;	/*!< Count of os_waits. May not be accurate */
	const char*	cfile_name;/*!< File name where lock created */
        /* last s-lock file/line is not guaranteed to be correct */
	const char*	last_s_file_name;/*!< File name where last s-locked */
	const char*	last_x_file_name;/*!< File name where last x-locked */
	ibool		writer_is_wait_ex;
				/*!< This is TRUE if the writer field is
				RW_LOCK_WAIT_EX; this field is located far
				from the memory update hotspot fields which
				are at the start of this struct, thus we can
				peek this field without causing much memory
				bus traffic */
	unsigned	cline:14;	/*!< Line where created */
	unsigned	last_s_line:14;	/*!< Line number where last time s-locked */
	unsigned	last_x_line:14;	/*!< Line number where last time x-locked */
#ifdef UNIV_DEBUG
	ulint	magic_n;	/*!< RW_LOCK_MAGIC_N */
/** Value of rw_lock_t::magic_n */
#define	RW_LOCK_MAGIC_N	22643
#endif /* UNIV_DEBUG */
};
```

变量 | 类型 |  说明
:----|:----|:----
lock_word | volatile lock_word_t | 标识当前latch状态
waiters | volatile ulint | 若wait array里有其他线程等待，设置为1
recursive | volatile ibool | 是否递归调用
writer_thread | volatile os_thread_id_t | 持有x-latch的线程ID
event | os_event_t | 用于等待队列
wait_ex_event | os_event_t | 用于通知x-latch等待线程
mutex | ib_mutex_t | 并发保护
list | UT_LIST_NODE_T(rw_lock_t) | 链表

lock_word状态包括：
- RW_LOCK_EX x-latch操作成功
- RW_LOCK_NOT_LOCKED x-latch操作不成功，之前已有线程持有s-latch
- RW_LOCK_WAIT_EX x-latch操作不成功，之前已有线程持有x-latch

```cpp
/* We decrement lock_word by this amount for each x_lock. It is also the
start value for the lock_word, meaning that it limits the maximum number
of concurrent read locks before the rw_lock breaks. The current value of
0x00100000 allows 1,048,575 concurrent readers and 2047 recursive writers.*/
#define X_LOCK_DECR		0x00100000

ulint
rw_lock_get_writer(
/*===============*/
	const rw_lock_t*	lock)	/*!< in: rw-lock */
{
	lint lock_word = lock->lock_word;
	if (lock_word > 0) {
		/* return NOT_LOCKED in s-lock state, like the writer
		member of the old lock implementation. */
		return(RW_LOCK_NOT_LOCKED);
	} else if ((lock_word == 0) || (lock_word <= -X_LOCK_DECR)) {
		return(RW_LOCK_EX);
	} else {
		ut_ad(lock_word > -X_LOCK_DECR);
		return(RW_LOCK_WAIT_EX);
	}
}
```

x-latch的操作由rw_lock_x_lock_func完成。跟mutex类似，函数首先调用rw_lock_x_lock_low通过CAS操作设置lock_word，如果不能获得x-latch，首先进行自旋操作，等待一段时间仍未能获得x-latch后线程放入wait array，等待被唤醒。x-latch支持递归操作，即允许同一线程多次获得x-latch，此时lock_word <= -X_LOCK_DECR。s-latch操作由rw_lock_s_lock_spin完成，过程基本相似，但是不支持递归调用。

## wait array

sync_array_t结构：
```cpp
/** Synchronization array */
struct sync_array_t {
	ulint		n_reserved;	/*!< number of currently reserved
					cells in the wait array */
	ulint		n_cells;	/*!< number of cells in the
					wait array */
	sync_cell_t*	array;		/*!< pointer to wait array */
	ib_mutex_t	mutex;		/*!< possible database mutex
					protecting this data structure */
	os_ib_mutex_t	os_mutex;	/*!< Possible operating system mutex
					protecting the data structure.
					As this data structure is used in
					constructing the database mutex,
					to prevent infinite recursion
					in implementation, we fall back to
					an OS mutex. */
	ulint		res_count;	/*!< count of cell reservations
					since creation of the array */
};
```

变量 | 类型 |  说明
:----|:----|:----
n_reserved | ulint | 已经使用cell数量
n_cells | ulint | 一共有多少cell
array | sync_cell_t* | cell数组
mutex | ib_mutex_t | InnoDB mutex（未使用）
os_mutex | os_ib_mutex_t | 使用操作系统mutex来保护该对象。由于wait array用来唤醒InnoDB mutex对象，因此使用操作系统自身mutex，避免可能存在的递归死锁情况
res_count | ulint |  被放到wait array的次数

sync_cell_t结构：
```cpp
/** A cell where an individual thread may wait suspended
until a resource is released. The suspending is implemented
using an operating system event semaphore. */
struct sync_cell_t {
	void*		wait_object;	/*!< pointer to the object the
					thread is waiting for; if NULL
					the cell is free for use */
	ib_mutex_t*	old_wait_mutex;	/*!< the latest wait mutex in cell */
	rw_lock_t*	old_wait_rw_lock;
					/*!< the latest wait rw-lock
					in cell */
	ulint		request_type;	/*!< lock type requested on the
					object */
	const char*	file;		/*!< in debug version file where
					requested */
	ulint		line;		/*!< in debug version line where
					requested */
	os_thread_id_t	thread;		/*!< thread id of this waiting
					thread */
	ibool		waiting;	/*!< TRUE if the thread has already
					called sync_array_event_wait
					on this cell */
	ib_int64_t	signal_count;	/*!< We capture the signal_count
					of the wait_object when we
					reset the event. This value is
					then passed on to os_event_wait
					and we wait only if the event
					has not been signalled in the
					period between the reset and
					wait call. */
	time_t		reservation_time;/*!< time when the thread reserved
					the wait cell */
};
```

变量 | 类型 |  说明
:----|:----|:----
wait_object | void* | 线程等待的latch，可以是mutex，也可以是rw-lock
old_wait_mutex | mutex_t* | cell等待的mutex
old_wait_rw_lock | rw_lock_t* | cell等待的rw-lock
thread | os_thread_id_t | 等待线程ID
waiting | ibool | 如果cell还在等待latch被释放，设置为TRUE
signal_count | ib_int64_t | 被唤醒次数
reservation_time | time_t | 线程放入到wait array的时间

wait array由多个cell组成，每个cell保存无法成功获得mutex或latch并等待被唤醒的线程。当线程不能立即持有latch时，会调用函数sync_array_reserve_cell分配wait array中的一个cell，之后等待的线程执行sync_array_wait_event进行休眠，等待被持有latch的线程唤醒。

注意os_event_wait_low函数中在休眠线程前有一个判断：
```cpp
	while (!event->is_set && event->signal_count == reset_sig_count) 	{
		os_cond_wait(&(event->cond_var), &(event->os_mutex));
	}
```

Thread A | Thread B |  Thread C
:----|:----|:----
os_event_reset() | |
 | os_event_set() [event->is_set == TRUE] |
 | | os_event_reset() [event->is_set == FALSE]
os_event_wait()  [infinite wait!] | |
 | | os_event_wait()  [infinite wait!]

如果上表所示，如果不加入检查，可能会出现无限等待的情况。每次调用os_event_reset，cell->signal_count记录线程等待latch的调用次数，并将其作为参数传给os_event_wait_low函数；而每次调用os_event_set()，event->signal_count加1。调用os_cond_wait陷入休眠前比较调用次数，即可以避免无限等待的场景。